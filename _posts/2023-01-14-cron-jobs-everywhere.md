---
layout: post
title: "Cron jobs, everywhere"
categories: [linux, macOS, powershell]
---

I have used [Kubernetes CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/){:target="_blank"} for multiple purposes. I have used [AWS EventBridge](https://docs.aws.amazon.com/lambda/latest/dg/services-cloudwatchevents-expressions.html){:target="_blank"} to schedule Lambda functions based on a schedule. I wrote several distinct [Spring schedulers](https://spring.io/guides/gs/scheduling-tasks){:target="_blank"} for various applications. And yet, I had never faced the need to use the father of them all: the UNIX cron job.

Well, there is a first time for everything. And just like Atahualpa and the Inca people before me, I had to create an actual cron job. Here are my notes on how to do it, on several different operating systems.

## Ubuntu

Cron jobs are defined in crontab files. The location of these crontab files depends on its type: user or system.

### user crontabs

To manage user crontab files, you use the `crontab` program directly:
  - `crontab -e` to create/edit it.
  - `crontab -l` to check its contents.
  - `crontab -r` to remove it.
  - Use the `-u {{USER}}` option to refer to a specific user's crontab.
  - All these files are located in `/var/spool/cron/crontabs`.

### system crontabs

To manage system crontab files, you need to handle the files directly.

The main crontab file is located at `/etc/crontab`, but additional crontab files can be put in the `/etc/cron.d` directory.

You may also have a few extra directories available: `cron.hourly`, `cron.daily`, `cron.weekly`, `cron.monthly`. They do precisely what you are thinking. Put a script in one of these folders if you want it to execute with the folder's frequency.

### Format

A crontab file is just a simple text file with a bunch of lines like this:

```text
* * * * * {USER} {COMMAND}
```

Each line defines a cron job.

The `{USER}` column is only required for system crontab files.

Cron also keeps track of the modification time on its crontab files, so no reload is necessary when they change. But if you have trust issues, reload it with `/etc/init.d/cron reload`.

I never quite remember what each column means, so I always use https://crontab.guru/ when writing one.

### Troubleshooting

By default, the cron service logs are stored in the `/var/log/syslog` global logging file. But you won't find the cron job's stdout and stderr there because the stdout/stderr for a cron job goes to the user's mailbox, in `/var/mail/{USER}`.

Inspecting the user's mailbox for a cron job is not ideal, so a usual pattern is to redirect the outputs of the cron job to a file, like:

```
* * * * * myuser /tmp/my-script.sh >> /tmp/cron-job.log
```

Also, if you want to ensure nothing goes to the mailbox, use `MAILTO=""` in your crontab file.

## MacOS

Cron is also available in macOS. There is a launchd daemon located at `/System/Library/LaunchDaemons/com.vix.cron.plist` which manages the cron daemon. Initially, I was confused by the daemon's name (`com.vix.cron`). But apparently, the version of cron we use today is known as the [Vixie cron](https://en.wikipedia.org/wiki/Cron#Modern_versions){:target="_blank"}, so that name makes sense, I guess.

Just put some cron job lines into a `/etc/crontab` file, and that launchd daemon will make sure your cron jobs run. You can't use the `cron.d` directory, though. You can also use the `crontab` CLI to manage the user's crontabs; those are located at `/usr/lib/cron/tabs/{USER}`.

## Windows

Windows don't have cron. But Windows does offer support for scheduled tasks. If using PowerShell, you can use the following cmdlets to create scheduled tasks:
  - [New-ScheduledTask](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/new-scheduledtask){:target="_blank"}
  - [New-ScheduledTaskTrigger](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/new-scheduledtasktrigger){:target="_blank"}
  - [New-ScheduledTaskAction](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/new-scheduledtaskaction){:target="_blank"}
  - [Register-ScheduledTask](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/register-scheduledtask?view=windowsserver2022-ps){:target="_blank"}

Here's an example:

```powershell
# Our scheduled task runs a powershell script, so we use the powershell command.
$action = New-ScheduledTaskAction `
  -Execute 'powershell.exe' `
  -Argument '-NonInteractive -NoLogo -NoProfile -ExecutionPolicy bypass -File "C:\test.ps1"'

# Run a task now, and repeat after 1 minute, forever.
$trigger = New-ScheduledTaskTrigger `
  -Once `
  -At (Get-Date) `
  -RepetitionInterval (New-TimeSpan -Minutes 1)

# The default one is good enough for most cases.
$settings = New-ScheduledTaskSettingsSet

# Create the scheduled task object and register it.
$task = New-ScheduledTask -Action $action -Trigger $trigger -Settings $settings
Register-ScheduledTask `
  -TaskName 'My useless scheduled task' `
  -InputObject $task `
  -User $UserName `
  -Password $Password
```

Alternatively, the [schtasks](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks){:target="_blank"} command can be used.

## Wrapping up

I had to figure out how to run these scheduled tasks on different types of machines to implement a custom health check for EC2 instances. My cron job was supposed to run every minute to guarantee that a specific binary was running on the EC2 instance and mark the instance as unhealthy if not. [This pull request](https://github.com/renderedtext/agent-aws-stack/pull/108){:target="_blank"} has the final solution.
