---
layout: post
title: "Where is my PATH, launchD?"
categories: [launchD, macOS]
---

While working on the MacOS support for [agent-aws-stack](https://github.com/renderedtext/agent-aws-stack){:target="_blank"}, I ran into a little issue with launchD. My scenario was: a Go binary executed as a launchD daemon. That daemon, upon termination, executes (in a non-login shell) a bash script:

```bash
token=$(curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 60" --fail --silent --show-error --location "http://169.254.169.254/latest/api/token")
instance_id=$(curl -H "X-aws-ec2-metadata-token: $token" --fail --silent --show-error --location "http://169.254.169.254/latest/meta-data/instance-id")
region=$(curl -H "X-aws-ec2-metadata-token: $token" --fail --silent --show-error --location "http://169.254.169.254/latest/meta-data/placement/region")

aws autoscaling terminate-instance-in-auto-scaling-group \
    --region "$region" \
    --instance-id "$instance_id" \
    --no-should-decrement-desired-capacity
```

All the script does is terminate the EC2 instance and remove it from the auto-scaling group.

In a Linux machine, as a systemD unit, the script executes successfully. However, in MacOS, as a launchD daemon, it doesn't. The script doesn't find the AWS CLI, exits with exit code 127, and displays that error message we all know and love:

```text
aws: command not found
```

The AWS CLI is located at `/usr/local/bin/aws`, but our PATH when the script is executed is `PATH=/usr/bin:/bin:/usr/sbin:/sbin`. For some reason, launchD is not including `/usr/local/bin` in the PATH it exposes to its daemons, as systemD does.

## The reason it works on systemD

You can see the environment systemD exposes to its system and user units using the `show-environment` command:

```bash
$ systemctl show-environment
LANG=C.UTF-8
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

$ systemctl --user show-environment
HOME=/home/ubuntu
LANG=C.UTF-8
LOGNAME=ubuntu
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin
SHELL=/bin/bash
SYSTEMD_EXEC_PID=1260
USER=ubuntu
XDG_RUNTIME_DIR=/run/user/1000
XDG_DATA_DIRS=/usr/local/share/:/usr/share/:/var/lib/snapd/desktop
DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus
```

Where does that environment comes from? From [environment generators](https://www.freedesktop.org/software/systemd/man/systemd.environment-generator.html){:target="_blank"}. SystemD comes with a [default one](https://www.freedesktop.org/software/systemd/man/environment.d.html){:target="_blank"} which loads the environment from some special places.

## What about launchD?

I was expecting launchD to have at least some environment setup for its daemons, but it doesn't. At least, not a default one, like systemD. There are a few different ways to configure it, though.

#### 1. In the .plist daemon configuration

In our launchD daemon configuration (located at `/Library/LaunchDaemons`), we can explicitly add the `PATH` variable we want:

```xml
<!-- rest of plist -->
<key>EnvironmentVariables</key>
<dict>
  <key>PATH</key>
  <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
</dict>
<!-- rest of plist -->
```

#### 2. Using launchctl setenv

From launchctl's man page:

```text
     setenv key value
              Specify an environment variable to be set on all future processes launched by launchd in the caller's context.
```

This would require restarting our daemon.

#### 3. Using launchctl config

From launchctl's man page:

```text
config system | user parameter value
              Sets persistent configuration information for launchd(8) domains. Only the system domain and user domains may be configured. The location
              of the persistent storage is an implementation detail, and changes to that storage should only be made through this subcommand. A reboot
              is required for changes made through this subcommand to take effect.

              Supported configuration parameters are:

              umask    Sets the umask(2) for services within the target domain to the value specified by value.  Note that this value is parsed by
                       strtoul(3) as an octal-encoded number, so there is no need to prefix it with a leading '0'.

              path     Sets the PATH environment variable for all services within the target domain to the string value.  The string value should
                       conform to the format outlined for the PATH environment variable in environ(7).  Note that if a service specifies its own PATH,
                       the service-specific environment variable will take precedence.

                       NOTE: This facility cannot be used to set general environment variables for all services within the domain. It is intentionally
                       scoped to the PATH environment variable and nothing else for security reasons.
```

This would require a reboot, though.

Note: previously (on MacOS versions < 10.10) you could use [/etc/launchd.conf](http://www.dowdandassociates.com/blog/content/howto-set-an-environment-variable-in-mac-os-x-slash-etc-slash-launchd-dot-conf/){:target="_blank"}, but that doesn’t work on newer versions.

## Conclusion

As the only thing I needed was to use the AWS CLI, I temporarily added the `/usr/local/bin` to the PATH in the script itself and [called it a day](https://github.com/renderedtext/agent-aws-stack/pull/105){:target="_blank"}.

In hindsight, I could've just done that in the first place, and see if the script would work. But, it's nice to understand what is going on before choosing to go with a simpler fix.

## Other resources

- [https://apple.stackexchange.com/questions/64916/defining-environment-variables-with-launchd-launchctl](https://apple.stackexchange.com/questions/64916/defining-environment-variables-with-launchd-launchctl)
- [https://superuser.com/questions/1649684/how-is-the-path-environment-variable-set-in-systemd-user-instance#:~:text=When systemd --user starts,%2Fuser-environment-generators%2F](https://superuser.com/questions/1649684/how-is-the-path-environment-variable-set-in-systemd-user-instance)