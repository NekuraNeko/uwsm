# Universal Wayland Session Manager

Experimental tool that wraps any standalone Wayland WM into a set of systemd units to
provide graphical user session with environment management, XDG autostart support, clean shutdown.

WIP(ish). Use at your onw risk.

The main structure of subcommands and features is more or less settled and will likely
not receive any drastic changes unless some illuminative idea comes by.
Nonetheless, keep an eye for commits with `[Breaking]` messages.

## Concepts and features

- Maximum use of systemd units and dependencies for startup, operation, and shutdown
  - binds to the basic [structure](https://systemd.io/DESKTOP_ENVIRONMENTS/#pre-defined-systemd-units) of `graphical-session-pre.target`, `graphical-session.target`, `xdg-desktop-autostart.target`
  - adds custom slices `app-graphical.slice`, `background-graphical.slice`, `session-graphical.slice` to put apps in and terminate them cleanly
  - provides convenient way of [launching apps to those slices](https://systemd.io/DESKTOP_ENVIRONMENTS/#xdg-standardization-for-applications)
- Systemd units are treated with hierarchy and universality in mind:
  - templated units with specifiers
  - named from common to specific where possible
  - allowing for high-level `name-.d` drop-ins
- WM-specific behavior can be added by plugins
  - currently supported: sway, wayfire, labwc
- Idempotently (well, best-effort-idempotently) handle environment:
  - On startup environment is prepared by:
    - sourcing shell profile
    - sourcing common `wayland-session-env` files (from $XDG_CONFIG_DIRS, $XDG_CONFIG_HOME)
    - sourcing WM-specific `wayland-session-env-${wm_id}` files (from $XDG_CONFIG_DIRS, $XDG_CONFIG_HOME)
  - Difference between environment state before and after preparation is exported into systemd user manager and dbus activation environment
  - On shutdown variables that were exported are unset from systemd user manager (dbus activation environment does not support unsetting, so those vars are emptied instead (!))
  - Lists of variables for export and cleanup are determined algorithmically by:
    - comparing environment before and after preparation procedures
    - boolean operations with predefined lists
- Can work with WM desktop entries from `wayland-sessions` in XDG data hierarchy
  - Desktop entry is used as WM instance ID
  - Data taken from entry (Can be amended or overridden via cli arguments):
    - `Exec` for argument list
    - `DesktopNames` for `XDG_CURRENT_DESKTOP` and `XDG_SESSION_DESKTOP`
    - `Name` and `Comment` for unit `Description`
  - Entries can be overridden, masked or added in `${XDG_DATA_HOME}/wayland-sessions/`
  - Optional interactive selector (requires whiptail), choice is saved in `wayland-session-default-id`
  - Desktop entry [actions](https://specifications.freedesktop.org/desktop-entry-spec/1.5/ar01s11.html) are supported
- Can run with arbitrary WM command line (saved as a unit drop-in)
- Better control of XDG autostart apps:
  - XDG autostart services (`app-*@autostart.service` units) are placed into `app-graphical.slice` that receives stop action before WM is stopped.
  - Can be mass-controlled via stopping and starting `wayland-session-xdg-autostart@${wm_id}.target`
- Try best to shutdown session cleanly via a net of dependencies between units
- Provide helpers for various operations:
  - finalizing service startup (WM service unit uses `Type=notify`) and exporting variables set by WM
  - launching applications as scopes or services in proper slices
  - checking conditions for launch at login (for integration into login shell profile)

## Installation

### 1. Executables and plugins

Put `wayland-session` executable somewhere in `$PATH`.

Put `wayland-session-plugins` dir somewhere in `${HOME}/.local/lib:/usr/local/lib:/usr/lib:/lib` (`UWSM_PLUGIN_PREFIX_PATH`)

### 2. Vars set by WM and Startup notification

Ensure your WM runs `wayland-session finalize` at startup:

- it fills systemd and dbus environments with essential vars set by WM: `WAYLAND_DISPLAY`, `DISPLAY`
- any other vars can be given as arguments by name
- any exported variables are also added to cleanup list
- if environment export is successful, it signals WM service readiness via `systemd-notify --ready`

Example snippet for sway config:

`exec exec wayland-session finalize SWAYSOCK I3SOCK XCURSOR_SIZE XCURSOR_THEME`

### 3. Slices

By default `wayland-session` launces WM service in `app.slice` and all processes spawned by WM will be
a part of `wayland-wm@${wm_id}.service` unit. This works, but is not an optimal solution.

Systemd [documentation](https://systemd.io/DESKTOP_ENVIRONMENTS/#pre-defined-systemd-units)
recommends running compositors in `session.slice` and launch apps as scopes or services in `app.slice`.

`wayland-session` provides convenient way of handling this.
It generates special nested slices that will also receive stop action ordered before
`wayland-wm@${wm_id}.service` shutdown:

- `app-graphical.slice`
- `background-graphical.slice`
- `session-graphical.slice`

`app-*@autostart.service` units are also modified to be started in `app-graphical.slice`.

To launch an app scoped inside one of those slices, use:

`wayland-session app [-s a|b|s|custom.slice] [-t scope|service] your_app [with args]`

Launching desktop entries is also supported:

`wayland-session app [-s a|b|s|custom.slice] [-t scope|service] your_app.desktop[:action] [with args]`

In this case args must be supported by the entry or its selected action according to
[XDG Desktop Entry Specification](https://specifications.freedesktop.org/desktop-entry-spec/1.5/ar01s07.html).

Example snippet for sway config on how to launch apps:

    # launch foot terminal by executable
    bindsym --to-code $mod+t exec exec wayland-session app foot
    
    # fuzzel has a very useful launch-prefix option
    bindsym --to-code $mod+r exec exec fuzzel --launch-prefix='wayland-session app' --log-no-syslog
    
    # launch SpaceFM via desktop entry
    bindsym --to-code $mod+e exec exec wayland-session app spacefm.desktop
    
    # featherpad desktop entry has "standalone-window" action
    bindsym --to-code $mod+n exec exec wayland-session app featherpad.desktop:standalone-window

When app launching is properly configured, WM service itself can be placed in `session.slice` by setting
environment variable `UWSM_USE_SESSION_SLICE=true` before generating units (best to export this
in `profile` before `wayland-session` invocation).
Or by adding `-S` argument to `start` subcommand.

Apps can also be launched as services by adding `-t service` argument or setting default
for `-t` via `UWSM_APP_UNIT_TYPE` env var.

## Operation

### Short story:

Start variants:

- `wayland-session start ${wm_id}`: generates and starts templated units with `@${wm_id}` instance.
- `wayland-session start ${wm_id} with "any complex" arguments`: also adds arguments for particular `@${wm_id}` instance.
- `-N, -[e]D, -C` can be used to add name, desktop names, description respectively.

If `${wm_id}` ends with `.desktop` or has a `.desktop:some-action` substring, `wayland-session` finds
desktop entry in `wayland-sessions` data hierarchy, uses Exec and DesktopNames from it
(along with name and comment for unit descriptons).

Arguments provided on command line are appended to the command line of desktop entry (unlike apps),
no argument processing is done (Please [file a bug report](https://github.com/Vladimir-csp/uwsm/issues/new/choose)
if you encounter any wayland-sessions desktop entry with %-fields).

If you want to customize WM execution provided with a desktop entry, copy it to
`~/.local/share/wayland-sessions/` and change to your liking, including adding [actions](https://specifications.freedesktop.org/desktop-entry-spec/1.5/ar01s11.html).

If `${wm_id}` is `select` or `default`, `wayland-session` invokes a menu to select desktop entries
available in `wayland-sessions` data hierarchy (including their actions).
Selection is saved, previous selection is highlighted (or launched right away in case of `default`).
Selected entry is used as `${wm_id}`.

There is also a separate `select` action (`wayland-session select`) that only selects and saves default `${wm_id}`
and does nothing else.

When started, `wayland-session` will hold while wayland session is running, and terminate session if
is itself interrupted or terminated.

To launch automatically after login on virtual console 1, if systemd is at `graphical.target`,
add this to shell profile:

    if wayland-session check may-start && wayland-session select
    then
    	exec wayland-session start default
    fi

`check may-start` checker subcommand, among other things, **screens for being in interactive login shell,
which is essential**, since profile sourcing can otherwise lead to nasty loops.

Stop with `wayland-session stop`.

### Longer story, tour under the hood:

`-h|--help` option is available for `wayland-session` and its subcommands.

#### Start and bind

(At least for now) units are generated by the script.

Run `wayland-session start -o ${wm}` to populate `${XDG_RUNTIME_DIR}/systemd/user/` with them and do
nothing else (`-o`).

Any remainder arguments are appended to WM argument list (even when `${wm}` is a desktop entry).
Use `--` to disambigue:

`wayland-session start -o ${wm} -- with "any complex" arguments`

Desktop entries can be overridden or added in `${XDG_DATA_HOME}/wayland-sessions/`.

Basic set of generated units:

- templated targets boud to stock systemd user-level targets
  - `wayland-session-pre@.target`
  - `wayland-session@.target`
  - `wayland-session-xdg-autostart@.target`
- templated services
  - `wayland-wm-env@.service` - environment preloader service
  - `wayland-wm@.service` - main WM service
- slices for apps nested in stock systemd user-level slices
  - `app-graphical.slice`
  - `background-graphical.slice`
  - `session-graphical.slice`
- tweaks
  - `wayland-wm-env@${wm}.service.d/custom.conf`, `wayland-wm@${wm}.service.d/custom.conf` - if arguments and/or various names were given on command line, they go here.
  - `app-@autostart.service.d/slice-tweak.conf` - assigns XDG autostart apps to `app-graphical.slice`

After units are generated, WM can be started by: `systemctl --user start wayland-wm@${wm}.service`

Add `--wait` to hold terminal until session ends.

`exec` it from login shell to bind to login session:

`exec systemctl --user start --wait wayland-wm@${wm}.service`

Still if login session is terminated, wayland session will continue running, most likely no longer being accessible.

To also bind it the other way around, shell traps are used:

`trap "if systemctl --user is-active -q wayland-wm@${wm}.service ; then systemctl --user --stop wayland-wm@${wm}.service ; fi" INT EXIT HUP TERM`

This makes the end of login shell also be the end of wayland session.

When `wayland-wm-env@.service` is started during `graphical-session-pre.target` startup,
`wayland-session aux prepare-env ${wm}` is launched (with shared set of custom arguments).

It runs shell code to prepare environment, that sources shell profile, `wayland-session-env*` files,
anything that plugins dictate. Environment state at the end of shell code is given back to the main process.
`wayland-session` is also smart enough to find login session associated with current TTY
and set `$XDG_SESSION_ID`, `$XDG_VTNR`.

The difference between initial env (that is the state of activation environment) and after all the
sourcing and setting is done, plus `varnames.always_export`, minus `varnames.never_export`, is added to
activation environment of systemd user manager and dbus.

Those variable names, plus `varnames.always_cleanup` minus `varnames.never_cleanup` are written to
a cleanup list file in runtime dir.

#### Startup finalization

`wayland-wm@.service` uses `Type=notify` and waits for WM to signal started state.
Activation environments will also need to receive essential variables like `WAYLAND_DISPLAY`
to launch graphical applications successfully.

`wayland-session finalize [VAR [VAR2...]]` runs:

    dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY [VAR [VAR2...]]
    systemctl --user import-environment WAYLAND_DISPLAY DISPLAY [VAR [VAR2...]]
    systemd-notify --ready

The first two together might be an overkill.

Only defined variables are used. Variables that are not blacklisted by `varnames.never_cleanup` set
are also added to cleanup list in runtime dir.

#### Stop

Just stop the main service: `systemctl --user stop "wayland-wm@${wm}.service"`, everything else will
stopped by systemd.

Wildcard `systemctl --user stop "wayland-wm@*.service"` will also work.

If start command was run with `exec` from login shell or `.profile`,
this stop command also doubles as a logout command.

When `wayland-wm-env@${wm}.service` is stopped, `wayland-session aux cleanup-env` is launched.
It looks for **any** cleanup files (`env_names_for_cleanup_*`) in runtime dir. Listed variables,
plus `varnames.always_cleanup` minus `varnames.never_cleanup`
are emptied in dbus activation environment and unset from systemd user manager environment.

When no WM is running, units can be removed (`-r`) by `wayland-session stop -r`.

Add WM to `-r` to remove only customization drop-ins: `wayland-session stop -r ${wm}`.

#### Profile integration

This example does the same thing as `check may-start` + `start` subcommand combination described earlier:
starts wayland session automatically upon login on tty1 if system is in `graphical.target`

**Screening for being in interactive login shell here is essential** (`[ "${0}" != "${0#-}" ]`).
`wayland-wm-env@${wm}.service` sources profile, which has a potential for nasty loops if run
unconditionally. Other conditions are a recommendation:

    MY_WM=sway
    if [ "${0}" != "${0#-}" ] && \
       [ "$XDG_VTNR" = "1" ] && \
       systemctl is-active -q graphical.target && \
       ! systemctl --user is-active -q wayland-wm@*.service
    then
        wayland-session start -o ${MY_WM}
        trap "if systemctl --user is-active -q wayland-wm@${MY_WM}.service ; then systemctl --user --stop wayland-wm@${MY_WM}.service ; fi" INT EXIT HUP TERM
        echo Starting ${MY_WM} WM
        systemctl --user start --wait wayland-wm@${MY_WM}.service &
        wait
        exit
    fi

## WM-specific actions

Shell plugins provide WM-specific functions during environment preparation.

Named `${__WM_BIN_ID__}.sh.in`, they should only contain specifically named functions.

`${__WM_BIN_ID__}` is derived from the item 0 of WM argv by applying `s/(^[^a-zA-Z]|[^a-zA-Z0-9_])+/_/`

It is used as plugin id and suffix in function names.

Variables available to plugins:
  - `__WM_ID__` - WM ID, effective first argument of `start`.
  - `__WM_BIN_ID__` - processed first item of WM argv.
  - `__WM_DESKTOP_NAMES__` - `:`-separated desktop names from `DesktopNames=` of entry and `-D` cli argument.
  - `__WM_FIRST_DESKTOP_NAME__` - first of the above.
  - `__WM_DESKTOP_NAMES_EXCLUSIVE__` - (`true`|`false`) `__WM_DESKTOP_NAMES__` came from cli argument and are marked as exclusive.

Functions available to plugins:
  - `load_config_env` - sources `$1` files from config hierarchy.
  - `load_wm_env` - standard function that loads `wayland-session-env-${__WM_ID__}` files from config hierarchy.

Functions that can be added by plugins:
  - `quirks_${__WM_BIN_ID__}` - called before env loading.
  - `load_wm_env_${__WM_BIN_ID__}` - replaces env loading. `load_wm_env` can be called inside to combine standard and custom loading.

Example:
    #!/bin/false

    # function to make arbitrary actions before loading wayland-session-env-${__WM_ID__}
    quirks_my_cool_wm() {
      # here additional vars can be set or unset
      export I_WANT_THIS_IN_SESSION=yes
      unset I_DO_NOT_WANT_THAT
      # or prepare a config for WM
      # or set a var to modify what sourcing wayland-session-env, wayland-session-env-${__WM_ID__}
      # in the next stage will do
      ...
    }

    load_wm_env_my_cool_wm() {
      # custom mechanism for loading of env (or a stub)
      # completely replaces loading from wayland-session-env-${__WM_ID__} in config dirs
      # so repeat it explicitly
      load_wm_env
      # and add ours
      load_config_env "${__WM_ID__}/env"
    }

## Compliments

Inspired by and adapted some techniques from:

- [sway-services](https://github.com/xdbob/sway-services)
- [sway-systemd](https://github.com/alebastr/sway-systemd)
- [sway](https://github.com/swaywm/sway)
- [Presentation by Martin Pitt](https://people.debian.org/~mpitt/systemd.conf-2016-graphical-session.pdf)
