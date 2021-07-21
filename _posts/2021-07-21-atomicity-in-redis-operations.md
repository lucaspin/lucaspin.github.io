---
layout: post
title: "Atomicity in Redis operations"
categories: [Redis, Databases]
---

When working with databases, atomicity is always something to have in mind. I recently started working with Redis again and came across a little problem I found worth sharing over here.

## Problem

You need to store elements in a list in Redis. More specifically, you need to append elements to the end of a list using an index. if the list length is above the index, you trim it before appending. if not, you just append to it.

There's not a built-in [command](https://redis.io/commands#list){:target="_blank"} to perform that operation available, so let's define an `Append()` function. I'll be using Golang and [this redis library](https://github.com/go-redis/redis) to do it:

```golang
func Append(client *redis.Client, key string, elements []string, index int64) error {
  length, _ := client.LLen(context.TODO(), key).Result()
  if length > index {
    _, err := client.LTrim(context.TODO(), key, 0, index-1).Result()
    if err != nil {
      return err
    }
  }

  _, err := client.RPush(context.TODO(), key, elements).Result()
  if err != nil {
    return err
  }

  return nil
}
```

Note that the `Append()` operation has 3 sub-operations in it: `LLen()`, `LTrim()`, `RPush()`. Each sub-operation is atomic in itself and contains a whole trip from the application running the code to the Redis cluster. But there is a problem with this approach: the `Append()` operation isn't atomic, even though its sub-operations are, which can lead to a few synchronization issues. For example, consider this code:

```golang
var INITIAL_VALUES = []string{"1", "2", "3", "4", "5"}
var ELEMENTS_TO_APPEND = []string{"4", "5"}

// create client
ctx := context.TODO()
client := redis.NewClient(&redis.Options{
  Addr:     "0.0.0.0:6379",
  Username: "",
  Password: "",
  DB:       0,
})

// initialize key
key := uuid.NewString()
log.Printf("Initializing %s", key)
client.RPush(ctx, key, INITIAL_VALUES).Result()

// call Append() twice right about the same time
go Append(client, key, ELEMENTS_TO_APPEND, 3) // #1
go Append(client, key, ELEMENTS_TO_APPEND, 3) // #2

// wait for both Append() calls to finish and check result
time.Sleep(time.Second)
elements := client.LRange(ctx, key, 0, -1).Val()
log.Printf("Final elements: %v", elements)
```

If we run that a few times, we can see what's wrong. To be honest, I thought it would take a decent amount of runs, but it looks like the race condition gods like me:

```sh
» go run main.go
2021/07/21 21:08:41 Initializing 256ceca4-180f-4746-a568-2b0d384038cf
2021/07/21 21:08:43 Final elements: [1 2 3 4 5]
» go run main.go
2021/07/21 21:08:46 Initializing 34dacaa7-127d-45ca-8b47-78fd74ca01ee
2021/07/21 21:08:48 Final elements: [1 2 3 4 5 4 5]
```

`[1 2 3 4 5 4 5]`? That's not correct. How did we end up with that? Let's break the operations down:

<image src="/public/img/redis-bad-operations.png"></image>

## Making it atomic

The usual way of solving the problem of accessing/modifying a shared resource is using locks or transactions. In Redis, if we were to follow that path, there are a few things to look at:
1. Using SETNX: this is the historical way of doing it, but this pattern is well-explained [here](https://redis.io/commands/setnx#design-pattern-locking-with-codesetnxcode){:target="_blank"} and [here](https://redis.io/commands/set#patterns){:target="_blank"} though, so I won't go into it here.
2. Using Redlock: to be quite honest, I started reading about Redlock but ended up leaving it, since I found another simpler approach. But here is [their docs](https://redis.io/topics/distlock){:target="_blank"} on it, [Martin Kleppmann's review](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html){:target="_blank"} of the algorithm and also the [author's response](http://antirez.com/news/101){:target="_blank"} to the review. It's quite extensive, but hey, you might be looking for something to read.

These solutions are mostly fine, but I wanted something simpler and didn't really want to rely on locks to solve this problem. And here is where the hero of this story emerges.

## Lua scripts

Redis allows you to execute lua scripts and [guarantees atomicity when executing them](https://redis.io/commands/eval#atomicity-of-scripts){:target="_blank"}. Putting in other words, a lua script is a way of wrapping multiple instructions into an atomic operation in Redis, without having to deal with locks or transactions.

Let's create a little lua script to do what we want and use [EVAL](https://redis.io/commands/eval){:target="_blank"} to execute it:

```golang
var APPEND_SCRIPT = `
local length = redis.call('LLEN', KEYS[1])
local upto = tonumber(ARGV[1])
if length > upto then
  redis.call('LTRIM', KEYS[1], 0, (upto - 1))
end
local elements = {unpack(ARGV)}
table.remove(elements, 1)
return redis.call('RPUSH', KEYS[1], unpack(elements))
`

func AppendWithScript(client *redis.Client, key string, elements []string, index int64) error {
  keys := []string{key}
  arguments := buildEvalArguments(elements, index)
  _, err := client.Eval(context.TODO(), APPEND_SCRIPT, keys, arguments).Result()
  if err != nil {
    return err
  }

  return nil
}

func buildEvalArguments(logs []string, index int64) []interface{} {
  arguments := make([]interface{}, len(logs)+1)
  arguments[0] = index
  for i, v := range logs {
    arguments[i+1] = v
  }
  return arguments
}
```

## What about performance?

Ok, this seems great and all, but what about my performance? Well, let's write some benchmarks for our functions:

```golang
func Benchmark__Append(b *testing.B) {
  for i := 0; i < b.N; i++ {
    Append(client, key, ELEMENTS_TO_APPEND, 3)
  }
}

func Benchmark__AppendWithScript(b *testing.B) {
  for i := 0; i < b.N; i++ {
    AppendWithScript(client, key, ELEMENTS_TO_APPEND, 3)
  }
}
```

And this is the result we get if we run them against a local redis instance running inside a docker container:

```
» go test -bench=.
goos: darwin
goarch: amd64
pkg: redis-playground
cpu: Intel(R) Core(TM) i7-6700HQ CPU @ 2.60GHz
Benchmark__Append-8             	     282	   4323202 ns/op
Benchmark__AppendWithScript-8   	     712	   1556933 ns/op
PASS
ok  	redis-playground	3.124s
```

`Append()` executed has an average speed of ~4.32ms while `AppendWithScript()` executes at ~1.55ms, ~278% times faster.

## Pre-loading lua scripts

Even though it is fast enough for our case, there's one thing that might hurt your performance when using lua scripts. Note that are sending the whole script every time to Redis and Redis is recompiling it every time. But the script is always the same.

Considering that, another thing you can do is preload the script before executing it in Redis using [SCRIPT LOAD](https://redis.io/commands/script-load){:target="_blank"} and [EVAL SHA](https://redis.io/commands/evalsha){:target="_blank"}:

```golang
func LoadScript(client *redis.Client) (string, error) {
	scriptSha, err := client.ScriptLoad(context.TODO(), APPEND_SCRIPT).Result()
	if err != nil {
		return "", err
	}

	return scriptSha, nil
}

func AppendWithPreloadedScript(client *redis.Client, scriptSha string, key string, elements []string, index int64) error {
	keys := []string{key}
	arguments := buildEvalArguments(elements, index)
	_, err := client.EvalSha(context.TODO(), scriptSha, keys, arguments).Result()
	if err != nil {
		return err
	}

	return nil
}
```

Running the tests again:

```
» go test -bench=.
goos: darwin
goarch: amd64
pkg: redis-playground
cpu: Intel(R) Core(TM) i7-6700HQ CPU @ 2.60GHz
Benchmark__Append-8                      	     268	   4288026 ns/op
Benchmark__AppendWithScript-8            	     710	   1555603 ns/op
Benchmark__AppendWithPreloadedScript-8   	     746	   1532615 ns/op
PASS
ok  	redis-playground	4.381s
```

Ok, so not a whole lot faster.

It's important to note here though that I am running these tests against a local redis instance running inside a docker container in my own machine, so there's no network latency in place. Also, the script we're using is not that big and doesn't consume a lot of bandwidth, but if you have a large script, pre-loading it might really help.

And that's what I had for today. Thanks for reading!
