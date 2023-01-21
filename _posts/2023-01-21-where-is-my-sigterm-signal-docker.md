---
layout: post
title: "Where is my SIGTERM, Docker?"
categories: [docker, kubernetes]
---

This issue has robbed me of a good half hour this week. And this is not the first time that has happened. Previously, I just fixed it and moved on. This time, I'll write about it here, and hopefully, that'll make it stick in my brain.

## The problem

I was running the [Semaphore agent](https://github.com/semaphoreci/agent){:target="_blank"} in a Kubernetes deployment. To do so, I had to put the agent in a Docker container. Here's what my Dockerfile looked like:

```dockerfile
FROM ubuntu:20.04

# ... a lot of commands not relevant to the issue at hand ...
# Copy the agent binary to the container
COPY build/agent /app/agent

# Start the container using the agent binary
CMD /app/agent start --config-file /app/config/semaphore-agent.yml
```

When the Kubernetes deployment is created, the agent starts just fine.

In an ideal world, when the agent is interrupted, it should receive a SIGTERM signal and then disconnect from the API before completely shutting down. So, when deleting the Kubernetes deployment, that's what I was expecting to happen.

But that didn't happen. The agent didn't get any signals and continued its happy life without disruptions. That pod remained in the `Terminating` state for some time until the [terminationGracePeriodSeconds](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination){:target="_blank"} was reached. Then, Kubernetes would send it a SIGKILL and move on.

The problem was that the agent needed that SIGTERM to clean up after itself properly. And it didn't. Where did that signal go?

## Hunting a SIGTERM

I was pretty confident that the issue was not with the Semaphore agent. I looked at the Kubernetes configuration, and everything looked alright. I had run into this issue before, so I had a hunch that the issue was in the Dockerfile.

I looked at it but couldn't find anything wrong with it. Alright, let's go to the [docs](https://docs.docker.com/engine/reference/builder/#cmd){:target="_blank"}:

> The CMD instruction has three forms:
> - CMD ["executable","param1","param2"] (exec form, this is the preferred form)
> - CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
> - CMD command param1 param2 (shell form)

I laughed when I read that. I remembered going into the same docs to find the root cause of the same issue before. The sentence "*this is the preferred form*" was almost like a punch in my stupid face.

## One last time, for the whole class

Docker has different forms for the CMD directive: a shell form and an exec form. In fact, those forms are present in other directives too, like [RUN](https://docs.docker.com/engine/reference/builder/#run){:target="_blank"} and [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint){:target="_blank"}.

The main difference between the two forms is that for shell form commands, the command runs in a shell (`sh -c <command>`). That means that if you are using the shell form to run another executable, the shell process will create a sub-process to run that executable. But the primary process in the container will still be the shell process.

If you look at the processes in the container, this is what you see:

```text
USER  PID  COMMAND
root    1  /bin/sh -c /app/agent start --config-file /app/config/semaphore-agent.yml
root    6  /app/agent start --config-file /app/config/semaphore-agent.yml
```

## Finding the SIGTERM

When Kubernetes sends a SIGTERM to the container, the shell is the process receiving it, not the executable. And the shell doesn't propagate that signal to the executable. To guarantee that the executable is the process receiving any signals coming from Kubernetes, the exec form needs to be used:

```patch
-CMD /app/agent start --config-file /app/config/semaphore-agent.yml
+CMD ["/app/agent" "start" "--config-file" "/app/config/semaphore-agent.yml"]
```

Now, if you look at the processes in the container, there's no additional `sh -c` process:

```text
USER  PID  COMMAND
root    1  /app/agent start --config-file /app/config/semaphore-agent.yml
```

## Friendly reminder

When you run into an issue, you will likely run into the same issue again in the future. If you just fix it and move on, it is also very likely that you will not remember how to fix it once you see it again.

This holds for programming, but I suspect it may hold for other parts of life as well. When finding the solution to a problem, it is always a good idea to document it. Your future self (and possibly other people) will thank you.
