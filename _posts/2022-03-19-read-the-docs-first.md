---
layout: post
title: "Read the documentation first"
categories: [powershell, encoding, utf-8]
---

The following story contains high quantities of sadness (or humor, I guess). It displays my stubbornness to read a piece of documentation while looking for a quick solution to my problem. I decided to document it here to remind myself of this. And since we all learn better with a story, here's mine.

## The problem

I have been working with Windows and PowerShell for a few weeks now. It was part of a need for one of our projects to support Windows. I had never touched Windows before, so a few challenges arose from that little journey, the last one (so far) being the one that surprisingly reverberated with me the most, and it had nothing to do with Windows and PowerShell at all.

Here is what I needed to do:

1. Write PowerShell commands in a file
2. Execute that file with PowerShell, using Go's `exec.Command`

It is straightforward. Except for one issue: I needed to support UTF-8 encoded characters, and that was not working. For example, if the file had this command to be executed:

```
Write-Host "特定の伝説に拠る物語の由来については諸説存在し。特定の伝説に拠る物"
```

The result was this:

```
ç‰¹å®šã®ä¼èª¬ã«æ‹ ã‚‹ç‰©èªžã®ç ±æ¥ã«ã¤ãã¦ã¯è«¸èª¬å˜åœ¨ã—ã€‚ç‰¹å®šã®ä¼èª¬ã«æ‹ ã‚‹ç‰©
```

## Show me the code

This is the function I was using:

```go
func createFile(filePath, content string) error {
  file, err := os.OpenFile(filePath, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644)
  if err != nil {
    return err
  }

  defer file.Close()

  _, err = file.Write([]byte(content))
  if err != nil {
    return err
  }

  return nil
}
```

It creates a file with some content. If you ever wrote or read Go code, you have seen this code thousands of times. Here's an example of how that function was being used:

```go
package main

import (
  "fmt"
  "os"
  "os/exec"
  "path/filepath"
)

func main() {
  fmt.Println(executeCommand())
}

func executeCommand() string {
  // create file with unicode characters
  path := filepath.Join(os.TempDir(), "myscript.ps1")
  _ = createFile(path, `Write-Host "特定の伝説に拠る物語の由来については諸説存在し。特定の伝説に拠る物"`)

  // execute file using powershell
  cmd := exec.Command("powershell", "-NoProfile", "-NonInteractive", "-File", path)
  output, err := cmd.CombinedOutput()
  if err != nil {
    fmt.Printf("Error executing script: %v\n", err)
  }

  return string(output)
}
```

## Investigation

I went straight to Google and started looking for things that could lead me to a solution. A few of those searches led to Stack Overflow answers which tangentially touched the issue. I started following those paths. After a few hours, none of them gave me a solution.

I was getting frustrated. Ok, stop. Let's rewind and start over. What's in the file?

```
user@windows-machine > type .\script.ps1
Write-Host "ç‰¹å®šã®ä¼èª¬ã«æ‹ ã‚‹ç‰©èªžã®ç”±æ¥ã«ã¤ã„ã¦ã¯è«¸èª¬å˜åœ¨ã—ã€‚ç‰¹å®šã®ä¼èª¬ã«æ‹ ã‚‹ç‰©"
```

What?! That's not what I wrote and that's not what I see in my MacOS machine (Windows is running in a vagrant box):

```
user@macos-machine > type script.ps1
Write-Host "特定の伝説に拠る物語の由来については諸説存在し。特定の伝説に拠る物"
```

Ok, MacOS understands the file one way, and Windows understands it another. What if I execute the same PowerShell command but without a file?

```
user@windows-machine > powershell -NoProfile -NonInteractive -Command 'Write-Host "特定の伝説に拠る物語の由来については諸説存在し。特定の伝説に拠る物"'
特定の伝説に拠る物語の由来については諸説存在し。特定の伝説に拠る物
```

We're onto something, but this doesn't make much sense. Why does PowerShell interpret the same file differently?

## Reading the documentation

I went to the [PowerShell documentation](https://docs.microsoft.com/en-us/powershell/){:target="_blank"}, searched for "encoding" and ended up at the [character encoding page](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_character_encoding){:target="_blank"}. I discovered something called the **byte-order-mark**, also known as the **BOM**, which is a signature that takes a few bytes at the start of the file. Its purpose is to let a reader know of a few essential things: the UTF format in use (UTF-8, UTF-16, or UTF-32) and the endianness, in the case of the latter two.

I also encountered this:

> Creating PowerShell scripts on a Unix-like platform or using a cross-platform editor on Windows, such as Visual Studio Code, results in a file encoded using UTF8NoBOM. These files work fine in PowerShell, but may break in Windows PowerShell if the file contains non-Ascii characters.

That sentence precisely describes what I was doing. It also explains why the issue I was seeing was happening:

> If you need to use non-Ascii characters in your scripts, save them as UTF-8 with BOM. Without the BOM, Windows PowerShell misinterprets your script as being encoded in the legacy "ANSI" codepage. Conversely, files that do have the UTF-8 BOM can be problematic on Unix-like platforms. Many Unix tools such as cat, sed, awk, and some editors such as gedit don't know how to treat the BOM.

This was my :face-palm: moment.

## Fixing the problem

After screaming in despair for a few moments, I fixed the problem. Since I was only concerned about UTF-8, I included the [UTF-8 BOM](https://docs.microsoft.com/en-us/globalization/encoding/byte-order-mark){:target="_blank"} in the file being executed, turning my `createFile()` function into this:

```go
func createFile(filePath, command string) error {
  file, err := os.OpenFile(filePath, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644)
  if err != nil {
    return err
  }

  defer file.Close()

  if runtime.GOOS == "windows" {
    file.Write([]byte{0xEF, 0xBB, 0xBF})
  }

  _, err = file.Write([]byte(command))
  if err != nil {
    return err
  }

  return nil
}
```

Issue is fixed. Everybody is happy. End of story.

## Conclusion

While this post documents a PowerShell behavior, which might be helpful again for that purpose in the future, I am writing this to remind myself (and whoever is reading) that reading the documentation should be the first thing you do when you encounter an issue.

Looking for answers in stack overflow can be helpful, but it can sometimes be a problem. It can be distractive and lead us into paths we shouldn't be pursuing at all. If I had gone to the documentation page first, I would've saved myself a few good hours.

Documentation is critical. Good documentation has all the context you need. Also, good documentation has search capabilities, so you don't need to read the whole thing or search it yourself for what you want.

What if the documentation is terrible? Well, then I guess you can search for other sources of information (or cry in the bathroom).
