---
title: "Sandboxing Applications with Bubblewrap: A Simple Script"
date: 2024-01-08
draft: false
---

Previously in this series, [we discovered how to use bubblewrap]({{<ref
"posts/sandboxing-1.md">}}) to sandbox simple applications. Then, we moved
on to [more complex applications]({{<ref "posts/sandboxing-2.md">}}),
and concluded that, while it works, the long command lines used were getting very
unwieldy.

I will now present you the script (unimaginatively called
[sandbox](https://gist.github.com/sloonz/4b7f5f575a96b6fe338534dbc2480a5d)) I use to sandbox my applications. Its configuration
file is located at `~/.config/sandbox.yml`.

It starts with **resources** : mostly path binds, but also environment
variables and D-Bus services. A **preset** is a named set of
resources. You can then associate presets to applications using **rules**.

Let’s start with a very basic preset, mirroring the very first command
we sandboxed, plus some basic things from the second part :

```yaml
presets:
  common:
    - args: [--clearenv, --unshare-pid, --die-with-parent, --proc, /proc, --dev, /dev, --tmpfs, /tmp, --new-session]
    - setenv: [PATH, LANG, XDG_RUNTIME_DIR, XDG_SESSION_TYPE, TERM, HOME, LOGNAME, USER]
    - ro-bind: /etc
    - ro-bind: /usr
    - args: [--symlink, usr/bin, /bin, --symlink, usr/bin, /sbin, --symlink, usr/lib, /lib, --symlink, usr/lib, /lib64, --tmpfs, "{env[XDG_RUNTIME_DIR]}"]
    - bind: /run/systemd/resolve
```

Nothing should surprise you here at this point. Let’s use this preset :

```shell
$ sandbox -p common zsh
$
```

Just after that, we created a per-application sandboxed home. Let’s make a preset to do that, too :

```
  private-home:
    - bind: ["{env[HOME]}/sandboxes/{executable}/", "{env[HOME]}"]
      bind-create: true
    - dir: "{env[HOME]}/.config"
    - dir: "{env[HOME]}/.cache"
    - dir: "{env[HOME]}/.local/share"
```

You can try it :

```shell
$ sandbox -p common -p private-home zsh
$
```

(note that all zsh instances will share the same sandboxed home)

Let’s define some more presets for desktop applications now. Again,
nothing surprising if you followed the previous posts (except we
don’t have to handle explicitly the `DBUS_SESSION_BUS_ADDRESS` environment variable or the
`xdg-dbus-proxy` socket : the script is handling that for us as soon as
we use a `dbus-*` resource) :

```yaml
  x11:
    - setenv: [DISPLAY]
    - ro-bind: /tmp/.X11-unix/
  wayland:
    - setenv: [WAYLAND_DISPLAY]
    - ro-bind: "{env[XDG_RUNTIME_DIR]}/{env[WAYLAND_DISPLAY]}"
  pulseaudio:
    - ro-bind: "{env[XDG_RUNTIME_DIR]}/pulse/native"
    - ro-bind-try: "{env[HOME]}/.config/pulse/cookie"
    - ro-bind-try: "{env[XDG_RUNTIME_DIR]}/pipewire-0"
  drm:
    - dev-bind: /dev/dri
    - ro-bind: /sys
  portal:
    - file: ["", "{env[XDG_RUNTIME_DIR]}/flatpak-info"]
    - file: ["", "/.flatpak-info"]
    - dbus-call: "org.freedesktop.portal.*=*"
    - dbus-broadcast: "org.freedesktop.portal.*=@/org/freedesktop/portal/*"
```

Now that we’re done with presets, we can define rules. Applications can be matched implicitly, from the name of the executable run, or explicitly. Let’s start with an implicit rule (based on the name of the executable), for example firefox :

```yaml
rules:
  - match:
      bin: firefox
    setup:
      - setenv:
          MOZ_ENABLE_WAYLAND: 1
      - use: [common, private-home, wayland, portal]
      - dbus-own: org.mozilla.firefox.*
      - bind: "{env[HOME]}/Downloads"
      - bind: ["{env[HOME]}/.config/mozilla", "{env[HOME]}/.mozilla"]
```

Now, running `sandbox firefox` will bind `~/Downloads` and setup
the common, private-home, wayland and portal presets. I also bind
its configuration (`~/.mozilla` in the sandboxed environment) to my
`~/.config/firefox` directory (because I try to keep my `~/sandboxes`
directory disposable, not keeping important stuff here).

Let’s present an explicit rule :

```yaml
  - match:
      name: shell
    setup:
      - use: [common, private-home]
```

This rule will never be matched automatically ; it has to be matched
manually by `sandbox --name shell zsh`. It will have its private home in
`~/.sandboxes/shell`. I tend to use it for CLI commands I want to run
sandboxed, but for which I don’t care about their persistent state
(and that I don't run often).

An interesting set of rules is the ones I use for node/npm, where I
want the command to be able to use the current directory, but pretty
much nothing else :

```yaml
  - match:
      bin: node
    setup:
      - use: [common, private-home]
      - bind-cwd: {}
      - cwd: true

  - match:
      bin: npx
    setup:
      - use: [common, private-home]
      - bind-cwd: {}
      - cwd: true

  - match:
      bin: npm
    setup:
      - use: [common, private-home]
      - bind-cwd: {}
      - cwd: true
```

That way, I can just run `sandbox npm ci` or `sandbox npm run ...`
in my current directory, and the stuff will just work (and be sandboxed).

You can also define a default, fallback rule :

```yaml
  - match:
      name: none

  # Fallback: anything else fall backs to a sandboxed empty home
  - setup:
    - use: [common, private-home, x11, wayland, pulseaudio, portal]
```

The `none` rule, is if you don't want to use the fallback for an unknown
command ; in that case you can do `sandbox --name none -p common --tmpfs
~ ...`

That’s pretty much it. One thing to remember is to use `--` to separate
`sandbox` arguments from the sandboxed arguments : `sandbox -- npm --save
install ...`

I present you this script as a simple Gist with only this blog post
as documentation. There is probably enough things to do here for an opportunity to grow a whole project
on the basis of this script. I won’t be the one doing it. If any
brave, motivated soul want to take up that task, go ahead, you have
my blessing. Everything I published here (on the blog, or the script)
is released under the CC-0 license.

If you don't want to use such a script, the closest alternative is probably [Firejail](https://github.com/netblue30/firejail). The key differences are :

* A lot of stuff that is coded as presets in the configuration file or in the script (like D-Bus management or the pulseaudio preset) is hard-coded in the main (suid) binary in firejail. I agree with the author of bubblewrap that it is a quite larger attack surface.

* Firejail comes with a lot of predefined rules. Most applications should work out of the box without having to work hard to make them work.

* There is no equivalent of raw `--bind` in Firejail, so if you want to do things like `--bind /opt/projects/my-project ~/workspace` or `--bind ~/.config/mozilla ~/.mozilla`, you will not able to do it.

* Firejail has some interesting additional features like per-sandbox firewall rules.

* If you go into a default sandbox with `firejail --no-profile zsh`,
you will see that the default sandbox in firejail allows everything —
you have to explicitly blacklist stuff. We saw the very first time
we launched bubblewrap that it takes the opposite approach — by default
nothing is shared, you have to explicitly whitelist stuff.
