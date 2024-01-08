---
title: "Sandboxing Applications with Bubblewrap: Securing a Basic Shell"
date: 2023-12-24
draft: false
---

[![XKCD #1200: Authorization](https://imgs.xkcd.com/comics/authorization.png)](https://xkcd.com/1200/)

Everybody knows that allowing different applications unlimited access
to each other's data is not exactly optimal from a security point of
view. While servers have enjoyed containers to isolate applications
from each other, we lack a good solution for the desktop. Or do we?

There is, obviously, [flatpak](https://github.com/flatpak/flatpak).
Unfortunately, flatpak present itself as a "Linux application sandboxing
and *distribution* framework". This will not do. I already have a
distribution. I’m pretty happy with it. I want to run my *distribution*
applications in a isolated maner.

Thankfully, the *sandboxing* part of flatpack
is actually a separate, lesser known project :
[bubblewrap](https://github.com/containers/bubblewrap). Let’s try to use it
to secure our desktop.

Let’s get started with one of the easiest things to sandbox, a shell :

```shell
$ bwrap zsh
bwrap: execvp zsh: No such file or directory
```

Uh… what?

Let’s go back to what bubblewrap is doing : it’s actually creating a
new, empty filesystem namespace. The keyword here is "empty". There’s
no zsh executable in it. Let’s fix this :

```shell
$ bwrap --ro-bind /usr /usr /usr/bin/zsh
bwrap: execvp /usr/bin/zsh: No such file or directory
```

Weird. But the following command will tell you what failed :

```shell
$ ldd /usr/bin/zsh
	linux-vdso.so.1 (0x00007fff5d189000)
	libcap.so.2 => /usr/lib/libcap.so.2 (0x00007ff55abe2000)
	libncursesw.so.6 => /usr/lib/libncursesw.so.6 (0x00007ff55ab6b000)
	libm.so.6 => /usr/lib/libm.so.6 (0x00007ff55aa7e000)
	libc.so.6 => /usr/lib/libc.so.6 (0x00007ff55a89c000)
	/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007ff55ad24000)
```

Yes, shared librairies. So we need `/lib64` too. To be sure, let’s also
include `/bin`, `/lib` and `/sbin`, although they are just symlinks on my
system and should not be needed. Let’s also add `/etc` for things like
`/etc/profile.d` or `/etc/localtime` :

```shell
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc /usr/bin/zsh
/usr/share/zsh/scripts/newuser:5: no such file or directory: /dev/null
zsh-newuser-install:23: no such file or directory: /dev/null
zsh-newuser-install:24: no such file or directory: /dev/null
$
```

Yeah, `/dev/null` is kinda important. Many applications will want it. We
could bind it (using `--dev-bind`), but then there’s also `/dev/zero`,
`/dev/urandom`, and probably others. We could bind `/dev`, but that means
sandboxed applications will have access to devices — this does not
sounds like a good idea. Thankfully, bubblewrap has our back and have
provided us with a `--dev` option (and a `--proc` option for similar
woes). We also have `--tmpfs` for `/tmp` Let’s use them :

```shell
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp /usr/bin/zsh
$ ls /
bin  dev  etc  lib  lib64  proc  sbin  tmp  usr
```

Notice the absence of `/home` : we didn't bind it, so it is not
accessible. Any compromised program that is run in this shell session will
be unable to access our personal data (absent any privilege escalation
exploit given root access).

Good! We’re sandboxed! We’re safe!

Or are we?

```shell
$ ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0  21592 12736 ?        Ss   10:37   0:01 /sbin/init verbose
root           2  0.0  0.0      0     0 ?        S    10:37   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        S    10:37   0:00 [pool_workqueue_release]
root           4  0.0  0.0      0     0 ?        I<   10:37   0:00 [kworker/R-rcu_g]
root           5  0.0  0.0      0     0 ?        I<   10:37   0:00 [kworker/R-rcu_p]
root           6  0.0  0.0      0     0 ?        I<   10:37   0:00 [kworker/R-slub_]
root           7  0.0  0.0      0     0 ?        I<   10:37   0:00 [kworker/R-netns]
root          12  0.0  0.0      0     0 ?        I<   10:37   0:00 [kworker/R-mm_pe]
...
systemd+     426  0.0  0.0  91220  8468 ?        Ssl  10:37   0:00 /usr/lib/systemd/systemd-timesyncd
avahi        438  0.0  0.0   8932  4776 ?        Ss   10:37   0:00 avahi-daemon: running [desk.local]
dbus         439  0.0  0.0   9752  4976 ?        Ss   10:37   0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only
root         441  0.0  0.0  11016  7284 ?        Ss   10:37   0:00 sshd: /usr/bin/sshd -D [listener] 0 of 10-100 startups
...
sloonz       1427  0.0  0.4 513896 70208 ?        Sl   10:38   0:13 /usr/lib/firefox/firefox -contentproc -parentBuildID 20231130105227 -prefsLen 44628 -prefMapSize 241694 -appDir /usr/lib/firefox/browser {5508672c-0163-4fa1-adeb-7f40773b136b} 3 true rdd
...
```

Uh oh…

```shell
$ env
...
SHELL=/bin/zsh
WORDCHARS=*?_-.[]~=&;!#$%^(){}<>
HISTSIZE=50000
I3SOCK=/run/user/1000/sway-ipc.1000.548.sock
SSH_AUTH_SOCK=/run/user/1000/ssh-agent.socket
CREDENTIALS_DIRECTORY=/run/credentials/getty@tty1.service
MEMORY_PRESSURE_WRITE=c29tZSAyMDAwMDAgMjAwMDAwMAA=
XCURSOR_SIZE=24
...
AWS_SECRET_ACCESS_KEY=[redacted]
```

Oh noes. Let’s sandbox harder.

```shell
$ bwrap --help
...
    --unshare-all                Unshare every namespace we support by default
    --share-net                  Retain the network namespace (can only combine with --unshare-all)
    --unshare-user               Create new user namespace (may be automatically implied if not setuid)
    --unshare-user-try           Create new user namespace if possible else continue by skipping it
    --unshare-ipc                Create new ipc namespace
    --unshare-pid                Create new pid namespace
    --unshare-net                Create new network namespace
    --unshare-uts                Create new uts namespace
    --unshare-cgroup             Create new cgroup namespace
    --unshare-cgroup-try         Create new cgroup namespace if possible else continue by skipping it
...
    --clearenv                   Unset all environment variables
...
```

What do we want to unshare here ?

Unsharing the network namespace is a terrible idea, unless you want
prevent an application access to any network (including localhost).

Unsharing the PID namespace (processes) seems a clear win, so does
clearing the environment.

The IPC namespace is *probably*, *most of the time* fine to unshare
(important things like fifo, pipes and unix sockets are on the filesystem
namespace, except abstract unix sockets which are on the network
namespace), but it’s also hard to see the point (the compromised process
would have to find an exploitable program running in the non-sandboxed
environment whose attack vector would be POSIX message queues or SYSV
IPC, which in practice are very rarely used by desktop applications). We
will see later that sandboxing graphical applications can come with some
complications, and unsharing the IPC namespace might bring up some
really tricky bugs to figure out on top of those. We will already have
enough on our plate when we will try to sandbox desktop apps, so let's
not unshare this.

I don’t see the point of unsharing UTS namespace (it’s about clearing
the hostname), same for cgroup (unless possibly if you want to apply
limits to the newly created cgroup later, but I never tried). I don’t
see any big deal unsharing them either. Toss a coin to decide.

The whole sharing or unsharing of user namespace thing is a (small but
annoying) can of worms that I won’t try to extensively cover here (if
ever). To make things short : it's affected by the way your distribution
has installed bubblewrap (suid or not) and will have minimal effects
(the biggest one being that unsharing means files belonging to root will
belong to nobody in the sandbox). Let’s be satisfied with the default
on your system (whichever it is).

So let’s add to `--clearenv` and `--unshare-ipc` to our baseline
bubblewrap arguments. If you’re feeling extra-paranoid, you can add
`--unshare-uts`, `--unshare-user` and `--unshare-ipc` :

```shell
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid /usr/bin/zsh
$ ls /
bin  dev  etc  lib  lib64  proc  sbin  tmp  usr
$ ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
sloonz          1  0.0  0.0   2720  1152 ?        S    15:39   0:00 bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid /usr/bin/zsh
sloonz          3  0.0  0.0   6084  4436 ?        S    15:39   0:00 /usr/bin/zsh
sloonz          6  100  0.0   8024  3988 ?        R+   15:39   0:00 ps aux
$ killall firefox
firefox: no process found
$ env
PWD=/
HOME=/home/sloonz
LOGNAME=sloonz
SHLVL=1
OLDPWD=/
_=/bin/env
```

It looks good for a stateless application (for example if you want to
sandbox `curl https://ipinfo.io` to get your IP). What if you want to
keep files between sessions ? Well, let’s use a temporary home :

```shell
$ mkdir ~/sandboxes/my-node-project
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/my-node-project ~ --chdir ~ /usr/bin/zsh
$ npm install whatever
```

That way, `node_modules` will be installed in `~` (within your sandbox) or
`~/sandboxes/my-node-project` (in the non-sandboxed environment). If you
happen to install a compromised node library, this won’t compromise
your home directory.

You may want to bind some common configuration files, like `~/.zshrc`
(unless you have a `AWS_SECRET_ACCESS_KEY` environment variable in
it) or `~/.config/nvim`. Remember to bind them readonly (`--ro-bind`
instead of `--bind`) ; otherwise a compromised process may write to them
a malicious payload to gain access the next time those files are read
(and executed) in your non-sandboxed environment.

[Next time]({{<ref "posts/sandboxing-2.md">}}), we’ll see the basics of running sandboxed graphical
applications like your IDE or your browser.
