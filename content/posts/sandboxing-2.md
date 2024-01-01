---
title: "Sandboxing Applications with Bubblewrap: Desktop Applications"
date: 2024-01-01
draft: false
---

[Last time]({{<ref "posts/sandboxing-1.md">}}), we discovered how to use
[bubblewrap](https://github.com/containers/bubblewrap) to sandbox simple
CLI applications. We will now try to sandbox desktop applications.

Desktop applications want access to a lot of different resources:
for example the Wayland (or X) server socket,
sound server socket or D-Bus services. You could grant blanket access to all such resources for every application, but that increases the attack
surface quite a lot. An alternative is to give access only to resources
used by the application you’re trying to sandbox — though figuring this out isn't always straightforward since nobody cares documenting the resources they are using.

# First step: A Wayland Application

As a start, in order to illustrate the kind of issues involved, let’s
just try to sandbox [foot](https://codeberg.org/dnkl/foot), a very simple
terminal emulator for Wayland, as if it were a CLI application :

```shell
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid foot
warn: main.c:437: 'C' is not a UTF-8 locale, using 'C.UTF-8' instead
Fontconfig error: No writable cache directories
[fontconfig error repeated multiple times]
Fontconfig error: No writable cache directories
error: XDG_RUNTIME_DIR is invalid or not set in the environment.
 err: wayland.c:1456: failed to connect to wayland; no compositor running?
```

There are a lot of errors to unpack here :

1. `warn: main.c:437: 'C' is not a UTF-8 locale, using 'C.UTF-8' instead`
: we’re clearing the environment variables. That includes the locale
(`$LANG`). CLI applications typically don’t mind, but desktop
applications typically do.

2. `Fontconfig error: No writable cache directories` : `fontconfig`
is a library used by a lot of desktop applications ; it wants to write
stuff in its cache directory `~/.cache/fontconfig`. Here, I don’t even
have a home, so it complains.

3. `error: XDG_RUNTIME_DIR is invalid or not set in the environment`
: desktop applications typically make heavy use of session services
; they usually reside in the path that the environment variable
`$XDG_RUNTIME_DIR` points to (which defaults to `/run/user/UID/`). We
also cleared that environment variable.

4. `err: wayland.c:1456: failed to connect to wayland; no compositor
running?` : Wayland applications have to connect to the Wayland
server, which is exposed on an Unix socket (in the `$XDG_RUNTIME_DIR`
directory). Without it, no Wayland application will be run.

Errors 1 and 3 are the same : some environment variables that the
typical CLI application don’t care about become important for desktop
applications. Thankfully, it is generally safe to just pass them to
the sandbox unchanged. Here is a list of environment variables that I
just pass as-is to the sandbox : `$HOME`, `$PATH`, `$LANG`, `$TERM`,
`$XDG_RUNTIME_DIR`, `$XDG_SESSION_TYPE`.

Error 2 occurs because there is no home directory in the sandbox.

Before fixing error 4, let’s fix errors 1-3. Also, starting from
now, I’ll asume that those environment variables are set in your
non-sandboxed environment ; if they are not, do so manually.

```shell
$ mkdir -p ~/sandboxes/foot
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/foot ~ --chdir ~ --setenv HOME "$HOME" --setenv PATH "$PATH" --setenv LANG "$LANG" --setenv TERM "$TERM" --setenv XDG_RUNTIME_DIR "$XDG_RUNTIME_DIR" --setenv XDG_SESSION_TYPE "$XDG_SESSION_TYPE" foot
 err: wayland.c:1456: failed to connect to wayland; no compositor running?
```

We now get to the meat of the subject : session/system resources. `foot`,
being a Wayland application, needs access to the Wayland server, using
the Unix socket that is located at `$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY`. We
could give a blanket access to `$XDG_RUNTIME_DIR` (by adding `--bind
"$XDG_RUNTIME_DIR" "$XDG_RUNTIME_DIR"`), but this command will reveal
that it is probably a bad idea :

```shell
$ ls $XDG_RUNTIME_DIR
at-spi  bus  dbus-1  dconf  doc  gnupg  nvim.56001.0  nvim.57140.0  p11-kit  pipewire-0  pipewire-0.lock  pipewire-0-manager  pipewire-0-manager.lock  pulse  ssh-agent.socket  ssh-cp  sway-ipc.1000.669.sock  systemd  wayland-1  wayland-1.lock
```

You really don’t want sandboxed applications to have access to your SSH
agent just because it has to be able to access to your Wayland server
(among others). Fine-grained decisions about what is accessible or not
are required. In our simple example, we will just need the Wayland server :

```shell
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/foot ~ --chdir ~ --setenv HOME "$HOME" --setenv PATH "$PATH" --setenv LANG "$LANG" --setenv TERM "$TERM" --setenv XDG_RUNTIME_DIR "$XDG_RUNTIME_DIR" --setenv XDG_SESSION_TYPE "$XDG_SESSION_TYPE" --setenv WAYLAND_DISPLAY "$WAYLAND_DISPLAY" --tmpfs "$XDG_RUNTIME_DIR" --ro-bind "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" foot
```

Our `bwrap` command is starting to get quite long ! But it works, and
the shell inside the sandboxed `foot` intance doesn’t have access to
our precious `$HOME` (or our SSH agent).

# Commonly Required Simple Resources

There are resources, like the Wayland server, which are easy to pass to
the sandboxed applications and required by quite a lot of applications. I
could give you an example for each one of them, but it would be quite
long. Instead, I will just list them, and how to pass them to the
sandboxed environment :

* The Wayland compositor : `--setenv WAYLAND_DISPLAY "$WAYLAND_DISPLAY" --ro-bind "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY"`

* The X Server : `--setenv DISPLAY "$DISPLAY" --ro-bind /tmp/.X11-unix /tmp/.X11-unix` 

* Pulseaudio/PireWire (sound server) : `--ro-bind "$XDG_RUNTIME_DIR/pulse/native" "$XDG_RUNTIME_DIR/pulse/native" --ro-bind-try ~/.config/pulse/cookie ~/.config/pulse/cookie --ro-bind-try "$XDG_RUNTIME_DIR/pipewire-0" "$XDG_RUNTIME_DIR/pipewire-0"`

* GPU (typically for games) : `--dev-bind /dev/dri /dev/dri --ro-bind /sys /sys`

* V4L (webcam) : `--dev-bind /dev/v4l /dev/v4l --dev-bind /dev/video0 /dev/video0`

* `resolved` (if you use systemd for domain name resolution) : `--ro-bind /run/systemd/resolve /run/systemd/resolve`

But of course, if I say "those are the easy one to pass to the
sandboxed enviroment", it means there are ones that are more tricky.

# D-Bus

The first (and biggest) tricky point is D-Bus. I won’t
explain to you what it is ; [Wikipedia has a wonderful page for
it](https://en.wikipedia.org/wiki/D-Bus). To make things short, it’s
a system allowing your applications to communicate with each other.

The easy way to deal with it is to just share it to the
sandboxed environment : `--setenv DBUS_SESSION_BUS_ADDRESS
"$DBUS_SESSION_BUS_ADDRESS" --ro-bind "$XDG_RUNTIME_DIR/bus"
"$XDG_RUNTIME_DIR/bus"`. It has the same issue as blanket-sharing
`$XDG_RUNTIME_DIR` : many applications offer you ways to control them via
D-Bus. Giving direct access to the session bus to a sandboxed application
means giving direct control to those applications. Yikes !

Another way, of course, is to not give access to D-Bus in the sandboxed
environment at all. Sometimes, it works. Often, the application will
just crash, or misbehave.

The flatpak developers have the same issue, and have come with a solution :
[xdg-dbus-proxy](https://github.com/flatpak/xdg-dbus-proxy), which is
essentially a D-Bus server which will proxy D-Bus traffic with a D-Bus
server, but with access control rules.

As an illustration, let’s try to sandbox Firefox, first without D-Bus :

```shell
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/firefox ~ --chdir ~ --setenv HOME "$HOME" --setenv PATH "$PATH" --setenv LANG "$LANG" --setenv TERM "$TERM" --setenv XDG_RUNTIME_DIR "$XDG_RUNTIME_DIR" --setenv XDG_SESSION_TYPE "$XDG_SESSION_TYPE" --setenv WAYLAND_DISPLAY "$WAYLAND_DISPLAY" --tmpfs "$XDG_RUNTIME_DIR" --ro-bind "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" firefox
```

Then, launch Firefox a second time (for example to visit github.com) :

```shell
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/firefox ~ --chdir ~ --setenv HOME "$HOME" --setenv PATH "$PATH" --setenv LANG "$LANG" --setenv TERM "$TERM" --setenv XDG_RUNTIME_DIR "$XDG_RUNTIME_DIR" --setenv XDG_SESSION_TYPE "$XDG_SESSION_TYPE" --setenv WAYLAND_DISPLAY "$WAYLAND_DISPLAY" --tmpfs "$XDG_RUNTIME_DIR" --ro-bind "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" firefox https://github.com
```

You would expect Firefox to open as a new tab in the existing (sandboxed)
Firefox instance. Instead, you get this error message :

> Firefox is already running, but is not responding. To use Firefox, you must first close the existing Firefox process, restart your device, or use a different profile.

What’s happening ? By design, we are using the same profile in both
cases, but with two different instances. Firefox does not support
this. Instead, when you run the `firefox` command, it starts by checking
if there is an existing running instance for that profile (by checking
a lock file), and if there is one, it will instruct it to open the given
URL in a new tab (via D-Bus). Since we share the profile files (including
the lock file), the second firefox command detects the existence of the
first instance, tries to open the URL in it, but can’t, because it
does not have access to D-Bus.

So now, let’s do the same experiment, but using xdg-dbus-proxy:

```shell
$ mkdir $XDG_RUNTIME_DIR/xdg-dbus-proxy
$ xdg-dbus-proxy  $DBUS_SESSION_BUS_ADDRESS $XDG_RUNTIME_DIR/xdg-dbus-proxy/firefox-main-instance.sock --filter --log --own=org.mozilla.firefox.* &
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/firefox ~ --chdir ~ --setenv HOME "$HOME" --setenv PATH "$PATH" --setenv LANG "$LANG" --setenv TERM "$TERM" --setenv XDG_RUNTIME_DIR "$XDG_RUNTIME_DIR" --setenv XDG_SESSION_TYPE "$XDG_SESSION_TYPE" --setenv WAYLAND_DISPLAY "$WAYLAND_DISPLAY" --tmpfs "$XDG_RUNTIME_DIR" --ro-bind "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" --setenv DBUS_SESSION_BUS_ADDRESS unix:path="$XDG_RUNTIME_DIR"/bus --bind "$XDG_RUNTIME_DIR"/xdg-dbus-proxy/firefox-main-instance.sock "$XDG_RUNTIME_DIR"/bus firefox
```

Here we give our first Firefox instance the right to own the names
`org.mozilla.firefox.*`, meaning acting as a D-Bus service.

```shell
$ xdg-dbus-proxy  $DBUS_SESSION_BUS_ADDRESS $XDG_RUNTIME_DIR/xdg-dbus-proxy/firefox-secondary-instance.sock --filter --log --talk=org.mozilla.firefox.* &
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/firefox ~ --chdir ~ --setenv HOME "$HOME" --setenv PATH "$PATH" --setenv LANG "$LANG" --setenv TERM "$TERM" --setenv XDG_RUNTIME_DIR "$XDG_RUNTIME_DIR" --setenv XDG_SESSION_TYPE "$XDG_SESSION_TYPE" --setenv WAYLAND_DISPLAY "$WAYLAND_DISPLAY" --tmpfs "$XDG_RUNTIME_DIR" --ro-bind "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" --setenv DBUS_SESSION_BUS_ADDRESS unix:path="$XDG_RUNTIME_DIR"/bus --bind "$XDG_RUNTIME_DIR"/xdg-dbus-proxy/firefox-secondary-instance.sock "$XDG_RUNTIME_DIR"/bus firefox https://github.com
```

Here we give our second Firefox instance the right to use the services
registered under the names `org.mozilla.firefox.*`. You should see a new tab opening in the first Firefox instance.

You should now know the basics of using D-Bus in a sandboxed
environment. Some closing remarks on this topic :

* `--own` implies `--talk`, so if you just want create a single command
for sandboxing Firefox, `--own` should suffice for both use cases
(primary and secondary instance).

* I used `--log` in the xdg-dbus-proxy, so you can look at what’s
going on. You can remove it once everyhing works to your satisfication.

* `xdg-dbus-proxy` supports more fine-grained permissions (for example,
only allowing to use the `OpenURL` method). If you’re interested in that
feature, read the documentation : trying it is left as an exercise to
the reader.

* I talked and demonstrated the session bus, but there is in fact two
D-Bus buses : the system bus (system-wide, owned by root) and the
session bus (session-wide, owned by the user owning the session). Most
desktop applications will use exclusively the session bus, but a
few will want access to the system bus (an important one is wine,
who makes use of the system D-Bus service `org.freedesktop.UDisks` or
`org.freedesktop.UDisks2`, depending on your wine version). The same
principles apply, and making it work is also left as an exercise to the
reader (seriously, this blog post is getting really long, and we aren’t
quite done yet).

* You can use `busctl --user` to introspect the session bus (drop `--user`
for the system bus).

# XDG Desktop Portal

[XDG Desktop Portal](https://flatpak.github.io/xdg-desktop-portal/) is
an attempt by flatpak (yes, them again. I don’t like the flatpak the
distribution solution, but you have to give credit to the guys behind the
project : they are doing solid work) to standardize access to resources
in a secure way, for example (but not limited to) :

* Taking screenshots and screencasts

* Bypassing the sandbox for the filesystem

* Accessing the webcam

* Opening URIs (files and resources)

I added securely, because the goal is to require user interaction for
sensitive operations. For example, XDG Desktop Portal does not give
insecure access to the whole filesystem : it allows sandboxed applications
to request access to a file outside of a sandbox with the mean of a file
picker. The user has to explicitly select the file. In other cases,
the user for example may have to allow certain actions.

It works by creating a D-Bus service in the non-sandboxed environment,
and giving access to `org.freedesktop.portal.*` (with `xdg-dbus-proxy`,
see previous section) to sandboxed applications.

First, let’s configure it. This is my configuration file :

```
$ cat ~/.config/xdg-desktop-portal/portals.conf
[preferred]
default=gtk
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

XDG Desktop Portal uses a system of backend to perform the
actions. Different host desktop environments will use different
backends. Here, I am using the gtk backend by default, and using the
wlr backend for screenshots and screencasts (because I’m using
sway). Refer to the documentation for different setups.

It should be started automatically with your session ; if not, `systemctl
--user start xdg-desktop-portal.service` should do the trick.

Let’s try it, sandboxing Firefox again and trying to open a file outside of
the sandbox.

```shell
$ touch ~/sandboxes/flatpak-info
$ xdg-dbus-proxy  $DBUS_SESSION_BUS_ADDRESS $XDG_RUNTIME_DIR/xdg-dbus-proxy/firefox-main-instance.sock --filter --log --own=org.mozilla.firefox.* --talk=org.freedesktop.portal.* &
$ bwrap --ro-bind /usr /usr --ro-bind /bin /bin --ro-bind /lib /lib --ro-bind /lib64 /lib64 --ro-bind /sbin /sbin --ro-bind /etc /etc --proc /proc --dev /dev --tmpfs /tmp --clearenv --unshare-pid --bind ~/sandboxes/firefox ~ --chdir ~ --setenv HOME "$HOME" --setenv PATH "$PATH" --setenv LANG "$LANG" --setenv TERM "$TERM" --setenv XDG_RUNTIME_DIR "$XDG_RUNTIME_DIR" --setenv XDG_SESSION_TYPE "$XDG_SESSION_TYPE" --setenv WAYLAND_DISPLAY "$WAYLAND_DISPLAY" --tmpfs "$XDG_RUNTIME_DIR" --ro-bind "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY" --setenv DBUS_SESSION_BUS_ADDRESS unix:path="$XDG_RUNTIME_DIR"/bus --bind "$XDG_RUNTIME_DIR"/xdg-dbus-proxy/firefox-main-instance.sock "$XDG_RUNTIME_DIR"/bus --bind ~/sandboxes/flatpak-info "$XDG_RUNTIME_DIR"/flatpak-info --bind ~/sandboxes/flatpak-info /.flatpak-info firefox
```

We had to create two empty files in the sandbox, `/.flatpak-info` and
`$XDG_RUNTIME_DIR/flatpak-info`, so Firefox can know we are in a sandbox
and should use the portal. Many applications are in the same cases.

Now, if you try to open a file (HTML file, or image) using Crtl-O,
you should see your whole, unsandboxed filesystem in the file chooser, and you should be able to open a file. However, if you visit the URL `file:///`, you should see that Firefox himself is still sandboxed. 

# Conclusion

You should now know almost everything there is to know to sandbox desktop
applications.

If an application gives you troubles, you can try the following :

* Use `busctl --user monitor` to look at what happens in the non-sandboxed
case, what names the application is trying to acquire, what D-Bus services
it is trying to access

* Use `strace -e file` to look at what files the application is trying
to access. Pay extra attention to `ENOENT` (file does not exists) errors

* Search for the application on the [flathub
repository](https://github.com/flathub/) ; you may find workarounds here
that the flatpak project had to use for that specific application

This concludes our overview of bubblewrap. However, the bwrap command is starting to get very unwiedly. Next week,
I’ll present you the script I actually use to keep it manageable.
