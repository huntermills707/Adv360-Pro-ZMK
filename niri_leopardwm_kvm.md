+++
title = "One Keyboard, Two Computers: Making Windows Feel Like niri"
date = 2026-10-29T09:00:00-07:00
draft = true
description = "Using F24 on a Kinesis Advantage360, keyd, LeopardWM, and AutoHotkey to keep one set of niri muscle memories across a KVM switch."
categories = ["homelab"]
tags = ["linux", "windows", "niri", "leopardwm", "autohotkey", "kinesis", "zmk", "qmk", "keyboards"]
author = "Hunter Mills"
slug = "niri-leopardwm-kvm"
+++

My personal computer runs Linux and [niri](https://github.com/niri-wm/niri). My work computer runs Windows, and Linux is not an option there. Both are attached to a KVM switch, so changing machines is a single button press.

The remaining friction is not the switch. It is the half second after it, when my hands still think they are in niri and Windows has other ideas.

I did not want to turn Windows into Linux. I wanted to preserve the small set of motions that make up a normal working session: focus a column, move a window, change workspaces, resize a column, and get back to the actual work. The same keyboard is already attached to both machines. It should not need two incompatible sets of muscle memory.

The answer was [LeopardWM](https://github.com/jcardama/LeopardWM), [AutoHotkey](https://www.autohotkey.com/), a Kinesis Advantage360, and one deliberately boring key: **F24**.

The hardware below is specifically the Advantage360. On the Pro, the stock programmable firmware is ZMK, so those are the primary examples; the QMK equivalents are included where they are useful. The same physical assignments can also be made in a supported graphical keymap editor.

## What “the same” needs to mean

LeopardWM is a good starting point because it is scroll-first, like niri. Windows live on a horizontal strip, and the monitor is a viewport into that strip. It is not trying to translate niri into a conventional binary tiling layout.

But matching the layout model is only half the job. The Windows bindings also need to match **my niri configuration**, not just niri's defaults. The important conventions are:

- `Mod+H/J/K/L` moves focus.
- `Mod+Ctrl+H/J/K/L` moves columns or windows.
- `Mod+Shift+H/L` focuses the adjacent monitor, while `Mod+Shift+J/K` changes workspaces.
- `Mod+1…9` selects a workspace.
- `Mod+F` toggles fullscreen.
- `Mod+Shift+T` toggles floating.
- `Mod+W` toggles tabs, `Mod+Q` closes a window, and `Mod+O` opens Overview.
- Application, lock, session, and monitor-power shortcuts retain the same physical chords.

The important word is **Mod**. On Linux, that usually means Super. On Windows, making it mean Super is where the trouble starts.

## The Mod problem

Three things with similar names need to stay separate:

| Name | What it actually is |
| --- | --- |
| niri `Mod` | A configuration alias, normally equivalent to `Super` |
| ZMK `&mo N` / QMK `MO(N)` | A firmware-layer shift, like the Kinesis Mod key |
| ZMK `LGUI` / QMK `KC_LGUI` | Left GUI, which Windows and Linux interpret as Win/Super |

A keyboard cannot transmit an abstract “niri Mod.” It can only transmit real keycodes. ZMK's `LGUI` and QMK's `KC_LGUI` produce that GUI modifier; QMK's `KC_LWIN` and `KC_LCMD` are aliases for the same thing.

That matters because Windows already owns a large collection of Win-key shortcuts. The obvious example is **Win+L**. niri uses `Mod+L` to focus the column on the right; Windows uses Win+L to lock the computer. If Mod emits GUI, those are not merely similar shortcuts. They are the same input event.

Moving the Super key to a more convenient position does not change that. Neither does spelling it `Super` instead of `Win` in a configuration file.

I could disable Windows locking, try to intercept system shortcuts, and audit every other Win-key collision. That is a lot of fragile machinery to recreate a problem I do not have on Linux. I would rather keep Win+L as a reliable way to lock the work machine.

## F24 becomes the boundary

The cleaner design is to give the window manager its own key:

```text
  Right-thumb Win key
            |
            v
           F24
            |
          KVM switch
       /            \
      /              \
 Linux               Windows
 keyd                LeopardWM
 F24 -> Super        F24 is a held modifier
      |                    |
    niri Mod           LeopardWM Mod
```

The keyboard sends F24 on both computers. What happens next is intentionally OS-specific:

- On **Linux**, [keyd](https://github.com/rvaiya/keyd) translates F24 to Left Meta/Super. niri can keep its ordinary `Mod=Super` configuration.
- On **Windows**, no translation happens. LeopardWM accepts F24 directly as a held modifier.
- The dedicated **④ key emits Left GUI**, giving both machines a real Win/Super key for operating-system shortcuts.

This gives me the same physical thumb key and the same HJKL chords without asking Windows to stop being Windows. `F24+L` moves focus on the work machine; **④+L** produces Win+L and locks it.

LeopardWM supports F13–F24 as modifiers. I picked F24 because it is even less likely than F13 to overlap an existing application shortcut.

PowerToys can map F13–F24 as individual keys, but its Shortcut Guide manifest cannot represent them as modifiers. LeopardWM therefore skips F24-modified bindings during that export; `lwm query hotkeys` still reports the effective configuration. PowerToys is not otherwise required for this setup.

## Firmware: F24 on Win, Super on ④

The Advantage360 needs only two key changes, and neither one costs an alphanumeric key:

| Physical key | New behavior | Purpose |
| --- | --- | --- |
| Right-thumb Win key | F24 | Window-management modifier on both computers |
| Dedicated ④ key | Left GUI | Real Win/Super for the operating system |

Everything else stays put, including the Kinesis Mod key. It still selects the firmware's Mod layer, so Bluetooth profile selection, the bootloader chords, and the other layer shortcuts behave exactly as before.

I chose the thumb Win key because that is already where my thumb goes for niri's `Mod`. On Linux, nothing about the physical motion changes: the key now sends F24, keyd turns it back into Super, and niri sees the same modifier it always has.

The circled ④ is one of the Advantage360's dedicated programmable hotkeys. It sits in the right inner column on the home row and is unassigned in the stock layout. It is not the keypad-layer 4 and does not occupy a normal typing-key position. Because it sits next to the original Win key, ④+L is an easy Win+L chord to remember.

On an Advantage360 Pro running the stock ZMK layout, the replacements are:

```c
&kp F24   // Right-thumb Win key (was &kp LGUI): held window-manager modifier
&kp LGUI  // Dedicated ④ key (was &none): actual Win/Super
```

These are substitutions in an existing keymap, not a complete ZMK `bindings` array. Apply them in every layer that explicitly assigns those positions. In the stock layout, that means both the base layer and the keypad layer. The Fn layer already uses `&trans` for the thumb key, so the thumb key inherits F24 there automatically. However, the Fn layer explicitly sets ④ to `&none`, so change that position to `&trans` if Super should also work while Fn is held. The Mod layer can stay as it is.

For a QMK firmware build, the equivalents are:

```c
KC_F24   // Right-thumb Win key
KC_LGUI  // Dedicated ④ key
```

A few details matter:

- **F24 must actually be reported.** The Adv360 Pro ZMK config enables NKRO and ships with `CONFIG_ZMK_HID_KEYBOARD_EXTENDED_REPORT=n` in `adv360_left_defconfig`. With that setting, F13–F24 are never sent. Set it to `y`; the left half is the central side that talks to the host. Kinesis documents this in its [Adv360 Pro firmware repository](https://github.com/KinesisCorporation/Adv360-Pro-ZMK).
- **Keep ④ as ordinary Left GUI.** Do not hard-code it to Win+L; a real modifier remains useful for every other OS shortcut.
- **Do not make the window-management key a layer-tap or hold-tap.** It should simply hold F24 while I press H/L.

On Windows, **④+L** is Win+L and locks the machine. On Linux, ④ is also plain Super, so ④+L acts like the thumb key and focuses the column to the right in niri. That is an intentional OS-specific difference, not another shared window-management binding. The AutoHotkey configuration below also reproduces my shared `Mod+Alt+L` lock chord.

The one new Linux dependency is keyd. Without it (on a bare TTY before the service starts, or on another Linux machine), the thumb key is just F24. ④ still sends real Super in that case, so Super is never lost.

## Translate F24 back to Super on Linux

The Linux side is deliberately small. I am not teaching niri a new modifier; I am translating the key before niri sees it.

A generic `/etc/keyd/default.conf` looks like this:

```ini
[ids]
*

[main]
f24 = leftmeta
```

Since my desktop runs NixOS, the declarative version belongs in `configuration.nix`:

```nix
services.keyd = {
  enable = true;
  keyboards.default = {
    ids = [ "*" ];
    settings.main.f24 = "leftmeta";
  };
};
```

The wildcard is acceptable here because the only change involves F24. For a more selective configuration, `sudo keyd monitor` will identify the keyboard, and its ID can replace `"*"`.

niri continues to use its existing bindings. The physical window-management key now arrives as Super, just like it did before the firmware change. In particular, my niri configuration already uses `Mod+O` for Overview, so Windows will use that same chord; no modifier-release binding is required.

## Install LeopardWM on Windows

On the work machine:

```powershell
winget install -e --id jcardama.LeopardWM
lwm run
```

The first run creates:

```text
%APPDATA%\leopardwm\config\config.toml
```

Back it up before editing:

```powershell
$cfg = "$env:APPDATA\leopardwm\config\config.toml"
Copy-Item $cfg "$cfg.backup"
notepad $cfg
```

I would not enable autostart yet. First make sure the KVM, F24, and recovery bindings all behave as expected.

## Configure the scrolling layout

Merge these settings into the generated configuration. Edit existing TOML sections rather than creating duplicates.

```toml
[layout]
# Match the niri layout: 16 px gaps and no outer struts.
gap = 16
outer_gap_left = 0
outer_gap_right = 0
outer_gap_top = 0
outer_gap_bottom = 0

width_presets = [0.33333, 0.5, 0.66667]
default_width_preset = 2
height_presets = [0.333, 0.5, 0.667]

# Closest match to niri's center-focused-column "never": scroll only
# enough to keep the focused column in view.
centering_mode = "just_in_view"
center_single_column = false

[behavior]
focus_new_windows = true
new_window_monitor = "focused"
new_window_placement = "new_column"

track_focus_changes = true
focus_follows_mouse = true
mouse_follows_focus = false

# Approximate niri's focus-window-or-workspace and
# move-window-or-to-workspace behavior at vertical edges.
workspace_edge_wrap = true
skip_empty_workspaces = false

[animation]
layout_duration_ms = 150
workspace_switch_duration_ms = 200
scroll_duration_ms = 200
overview_duration_ms = 150
easing = "ease_out"
reduce_motion_on_battery = true
```

`just_in_view` is the closest match to my niri `center-focused-column "never"` setting. `center` recenters after every navigation command, and `on_overflow` recenters only when a column is wider than the screen. `focus_follows_mouse = true` matches my niri input configuration; set it to false on both systems if that is not the desired behavior.

## The shared LeopardWM keymap

Replace LeopardWM's existing `[hotkeys]` section with this configuration. Here, **Mod means F24**.

```toml
[hotkeys]
scroll_modifier = "F24"

# Close, focus, and basic movement
"F24+Q" = "close_window"
"F24+H" = "focus_left"
"F24+L" = "focus_right"
"F24+K" = "focus_up"
"F24+J" = "focus_down"

"F24+Ctrl+H" = "move_column_left"
"F24+Ctrl+L" = "move_column_right"
"F24+Ctrl+K" = "move_window_up"
"F24+Ctrl+J" = "move_window_down"

# Monitor focus and vertical workspace navigation
"F24+Shift+H" = "focus_monitor_left"
"F24+Shift+L" = "focus_monitor_right"
"F24+Shift+K" = "workspace_prev"
"F24+Shift+J" = "workspace_next"

# Monitor and workspace movement
# H/L move the focused window rather than niri's whole column.
"F24+Ctrl+Shift+H" = "move_to_monitor_left"
"F24+Ctrl+Shift+L" = "move_to_monitor_right"
"F24+Ctrl+Shift+K" = "move_to_workspace_prev"
"F24+Ctrl+Shift+J" = "move_to_workspace_next"

# niri's consume-or-expel is one action; LeopardWM splits it. These only
# consume. expel_to_left/expel_to_right exist if a separate chord is wanted.
"F24+," = "consume_from_left"
"F24+." = "consume_from_right"

# Numbered workspaces
"F24+1" = "switch_workspace_1"
"F24+2" = "switch_workspace_2"
"F24+3" = "switch_workspace_3"
"F24+4" = "switch_workspace_4"
"F24+5" = "switch_workspace_5"
"F24+6" = "switch_workspace_6"
"F24+7" = "switch_workspace_7"
"F24+8" = "switch_workspace_8"
"F24+9" = "switch_workspace_9"

# These move the focused window, not niri's whole column.
"F24+Ctrl+1" = "move_to_workspace_1"
"F24+Ctrl+2" = "move_to_workspace_2"
"F24+Ctrl+3" = "move_to_workspace_3"
"F24+Ctrl+4" = "move_to_workspace_4"
"F24+Ctrl+5" = "move_to_workspace_5"
"F24+Ctrl+6" = "move_to_workspace_6"
"F24+Ctrl+7" = "move_to_workspace_7"
"F24+Ctrl+8" = "move_to_workspace_8"
"F24+Ctrl+9" = "move_to_workspace_9"

# Layout controls
# Ctrl+F is an approximation of expand-column-to-available-width.
# niri steps width/height by 10%; LeopardWM has no step resize, so
# -/= cycle through width_presets and height_presets instead.
"F24+Ctrl+F" = "maximize_column"
"F24+C" = "center_column"
"F24+-" = "cycle_width_down"
"F24+=" = "cycle_width_up"
"F24+Shift+-" = "cycle_height_down"
"F24+Shift+=" = "cycle_height_up"

# Modes and Overview
"F24+Shift+T" = "toggle_floating"
"F24+F" = "toggle_fullscreen"
"F24+W" = "toggle_tabbed"
"F24+O" = "toggle_overview"

# A WM escape hatch, not niri's shortcut-inhibitor toggle.
"F24+Escape" = "toggle_pause"

# Workspace-oriented substitutes for niriswitcher.
"Alt+`" = "toggle_overview"
"Alt+Shift+`" = "toggle_overview"

# Windows-side session management
"F24+Ctrl+Alt+R" = "reload"
"F24+Ctrl+Alt+Escape" = "panic_revert"
```

This deliberately leaves several chords unbound:

- `F24+Shift+V`: there is no clean floating-versus-tiled focus switch.
- `F24+Ctrl+C`: there is no center-visible-columns action.
- `F24+Shift+Y/U/I/O`: LeopardWM has no equivalent whole-workspace reorder or move operations.
- `F24+Tab`: LeopardWM has no previous-workspace action.

It also follows my config where it differs from stock niri: `F24+F` is fullscreen rather than maximize-column, and `F24+Shift+J/K` changes workspaces rather than monitors.

## Scrolling and touchpad navigation

This is an optional Windows-side enhancement rather than a shared niri binding:

```toml
[gestures]
enabled = true
swipe_left = "focus_left"
swipe_right = "focus_right"
swipe_up = "focus_up"
swipe_down = "focus_down"

# Mod + vertical wheel changes workspaces.
scroll_up = "workspace_prev"
scroll_down = "workspace_next"
```

For a Precision Touchpad that does not produce usable swipes through the default path, LeopardWM also has a native Raw Input mode:

```toml
[gestures]
raw_input = true
```

That mode takes effect after a full restart:

```powershell
lwm stop
lwm run
```

Windows' three- and four-finger touchpad gestures should be set to **Nothing** under **Settings → Bluetooth & devices → Touchpad** when using Raw Input. Otherwise Windows may consume the gesture before LeopardWM sees it. Compatibility varies, so `raw_input = false` remains the fallback.

## Fill the non-WM gaps with AutoHotkey

LeopardWM's hotkey catalog manages windows; it does not reproduce niri's general `spawn` bindings or Noctalia's shell integration. AutoHotkey v2 covers the remaining application, screenshot, lock, session, monitor-power, and help-overlay chords without adding another launcher or window manager.

The two programs have to cooperate at the keyboard-hook level, and the order matters:

- LeopardWM uses a low-level keyboard hook. Once any binding uses F24 as a modifier, that hook **swallows F24**. Nothing installed behind it ever sees the key.
- Windows calls the most recently installed low-level hook first.

So AutoHotkey has to sit **in front of** LeopardWM. It sees F24 and the chord, handles only the exact chords it owns, and passes everything else down to LeopardWM, including F24 itself.

Save the following as `niri-windows.ahk` and run it after installing AutoHotkey v2. Adjust executable names or paths for the work machine.

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force
#UseHook

; Keep AutoHotkey's hook in front of LeopardWM's. Windows calls the newest
; low-level hook first, so reinstall ours whenever a new LeopardWM process
; appears (login, or after lwm stop / lwm run).
lwmPid := 0
KeepHookFirst() {
    global lwmPid
    pid := ProcessExist("leopardwm.exe")
    if pid && pid != lwmPid {
        lwmPid := pid
        Sleep 1000
        InstallKeybdHook(true, true)
    }
}
KeepHookFirst()
SetTimer KeepHookFirst, 2000

; F24 held with no LeopardWM or AutoHotkey binding would otherwise type the
; bare letter into the focused app. Swallow keys neither program uses, so an
; unbound Mod chord does nothing, as in niri.
HotIf (*) => GetKeyState("F24", "P")
for key in ["a", "g", "i", "m", "n", "u", "v", "x", "y", "z", "Tab"]
    Hotkey "*" key, (*) => 0
HotIf

; Hotkeys without * match their modifiers exactly: t:: fires for Mod+T only,
; so Mod+Shift+T still reaches LeopardWM as toggle_floating.
#HotIf GetKeyState("F24", "P")

; Applications
t::Run "wt.exe"
d::Send "#r"   ; Windows Run dialog; not a Noctalia launcher clone
b::Run "firefox.exe"
p::Run "firefox.exe --new-window https://account.proton.me/apps"
e::Run "explorer.exe"

; Mod+S screenshot: the Snipping Tool region capture
s::Send "#+s"

; Mod+Alt+L locks Windows. Plain Mod+L still reaches LeopardWM.
!l::DllCall("user32\LockWorkStation")

; Mod+Shift+Q opens a session menu. Plain Mod+Q still closes a window.
+q::ShowSessionMenu()

; Mod+Shift+P turns the monitors off.
+p::MonitorOff()

; Mod+Shift+Esc opens help. Plain Mod+Esc still pauses LeopardWM.
+Esc::ShowHotkeyHelp()

#HotIf

MonitorOff() {
    ; Let the chord's keys come up first, or the key-up wakes the displays.
    KeyWait "F24"
    KeyWait "Shift"
    ; HWND_BROADCAST, WM_SYSCOMMAND, SC_MONITORPOWER, power off
    DllCall("user32\SendMessage", "ptr", 0xFFFF, "uint", 0x0112, "ptr", 0xF170, "ptr", 2)
}

ConfirmRun(prompt, command) {
    if MsgBox(prompt, "Confirm", "YesNo Icon?") = "Yes"
        Run(command)
}

ShowSessionMenu() {
    m := Menu()
    m.Add("Lock", (*) => DllCall("user32\LockWorkStation"))
    m.Add("Sign out", (*) => ConfirmRun("Sign out of Windows?", "shutdown.exe /l"))
    m.Add("Restart", (*) => ConfirmRun("Restart Windows?", "shutdown.exe /r /t 0"))
    m.Add("Shut down", (*) => ConfirmRun("Shut down Windows?", "shutdown.exe /s /t 0"))
    m.Show()
}

ShowHotkeyHelp() {
    text := "
    (
Mod = right-thumb key (F24)      Win = ④

Mod+H/J/K/L          Focus
Mod+Ctrl+H/J/K/L     Move column/window
Mod+Shift+H/L        Focus monitor
Mod+Shift+J/K        Change workspace
Mod+1...9            Select workspace
Mod+Ctrl+1...9       Move window to workspace
Mod+- / Mod+=        Cycle column width
Mod+Shift+- / =      Cycle window height
Mod+Shift+T          Toggle floating
Mod+F                Toggle fullscreen
Mod+W                Toggle tabbed column
Mod+O                Overview
Mod+Q                Close window
Mod+T/B/P/E/D        Terminal, browser, Proton, files, Run
Mod+S                Screenshot
Mod+Alt+L            Lock Windows
Mod+Shift+Q          Session menu
Mod+Shift+P          Turn monitors off
Mod+Shift+Esc        This help
Mod+Ctrl+Alt+Esc     LeopardWM emergency restore
    )"
    MsgBox(text, "LeopardWM / niri keys", 4096)
}
```

Hotkeys written without `*` only fire when the held modifiers match exactly, which is what keeps the two programs from fighting over a key. `F24+T` launches a terminal from AutoHotkey, while `F24+Shift+T` falls through to LeopardWM and toggles floating. The same applies to L, Q, P, and Escape. LeopardWM's own bindings are also exact-match, so `F24+Alt+L` never triggers its `F24+L` focus action.

The swallow list covers keys that neither program binds with F24. If I later bind one of them in LeopardWM, I have to remove it from that list. Otherwise AutoHotkey consumes the chord before LeopardWM sees it.

This is a small set of replacements, not a Windows port of Noctalia:

- `Mod+D` opens the Windows Run dialog rather than a full application launcher.
- `Mod+S` opens the Snipping Tool's region capture rather than niri's screenshot UI.
- `Mod+Shift+Q` opens an AutoHotkey menu rather than the Noctalia session menu.
- `Mod+Shift+Escape` opens a simple always-on-top help window.
- Standard Windows media keys can remain native; no AutoHotkey mapping is needed for ordinary volume, playback, next, and previous commands.

Run the script manually during testing. Once it works, place a shortcut to it in the Windows Startup folder. Startup order does not matter, because the script moves its hook back in front whenever LeopardWM starts.

## The KVM is part of the test environment

My switch is a KCEVE [KC-KVM402DH](https://www.kvm-switch.de/en/4-pc-dual-monitor-with-hdmi-4k120hz-and-dp-4k144hz-kvm-switch-with-4x-usb-3-0-kc-kvm402dh.html), a four-computer, dual-monitor (DisplayPort and HDMI) switch. Each computer connects to it with one USB 3.0 cable, and its front panel has four USB 3.0 ports for the keyboard, mouse, and anything else. According to its [manual](https://m.media-amazon.com/images/I/91jb8LA8T0L.pdf), it switches only with the front buttons or the wired remote. It has no keyboard hotkeys and no separate emulated keyboard or mouse ports.

That makes it a switched USB hub rather than an emulating KVM. The Advantage360's own USB connection moves from one computer to the other, and each computer talks directly to the real keyboard. For this setup, that is the friendly kind of switch:

- **F24 and NKRO pass through untouched.** Switches that emulate a keyboard sometimes drop F13–F24 or fall back to six-key reporting. A hub never interprets the keys at all.
- **No hotkeys to collide with.** The switch never listens for key sequences, so it cannot take any Mod chords for itself.

The trade-off is that every switch is a USB unplug and replug. The keyboard disappears from one computer and enumerates on the other, which takes a few seconds.

These are the checks I care about:

1. **Test F24 directly, then through the switch.** Plug the keyboard straight into each computer first, and only then through the KVM. With a hub-style switch the results should match. If F24 goes missing, check the firmware setting before suspecting the switch: with NKRO on, the Adv360 Pro needs `CONFIG_ZMK_HID_KEYBOARD_EXTENDED_REPORT=y`. `CONFIG_ZMK_HID_REPORT_TYPE_HKRO=y` also carries F24 without the extended report.
2. **Switch with keys released.** Because switching unplugs the keyboard, a key held during the switch may never deliver its key-up to the computer being left. LeopardWM tracks F24 from its own hook, so it could treat the modifier as stuck. Pressing the buttons or remote by hand makes this unlikely; if it happens, tap the thumb key once.
3. **Keep the keyboard on USB.** The Advantage360 Pro can also connect over Bluetooth. If it is paired with either computer, ZMK may send keystrokes over Bluetooth during the gap while USB re-enumerates, so they land on the wrong machine. The setup is cleanest with no active Bluetooth profile for either computer.
4. **Let keyd pick the keyboard back up.** The `*` wildcard in the keyd configuration grabs the keyboard again each time it re-enumerates on Linux. A device-specific ID works too, as long as it matches the keyboard and not the switch's hub.
5. **Keep recovery independent of the custom key.** Leave ④ available as Win/Super, know where LeopardWM's tray menu is, and remember that `lwm stop` works from an ordinary terminal.
6. **Test reconnects, not just startup.** Switch Linux → Windows → Linux, then repeat after one machine has slept. Only the selected computer has the keyboard, so only that computer can be woken from it. The everyday path is the one that needs to be reliable.
7. **Enable autostart last.** A cold boot is only one of the paths the setup needs to survive.

## Validate the Windows side

Reload LeopardWM and inspect its effective configuration:

```powershell
lwm reload
lwm query hotkeys
lwm doctor
```

Then run through a short smoke test:

- Open four or five ordinary applications.
- Use `F24+H/L` to move between columns and `F24+J/K` within a column.
- Move columns and windows with `F24+Ctrl+H/J/K/L`.
- Use `F24+Shift+J/K` to change workspaces.
- Move a window with `F24+Ctrl+1…9` and `F24+Ctrl+Shift+J/K`.
- Confirm that `F24+F` toggles fullscreen and `F24+Shift+T` toggles floating.
- Open Overview with `F24+O`.
- Launch apps with `F24+T/B/P/E/D` and take a screenshot with `F24+S`.
- In a text field, press an unbound chord such as `F24+X` and confirm that nothing is typed.
- Open the session menu with `F24+Shift+Q` and help with `F24+Shift+Escape`.
- Confirm that `F24+Alt+L` locks Windows; test **④+L** as the alternative.
- Test `F24+Shift+P` only when ready for the displays to switch off.
- On Linux, confirm that F24 still acts as niri Mod and ④ acts as ordinary Super.
- Test `F24+Ctrl+Alt+Escape` as the emergency restore binding.
- Run `lwm stop` and `lwm run`, wait a few seconds, then confirm that `F24+T` still opens a terminal and `F24+Shift+T` still toggles floating. This checks that AutoHotkey moved its hook back in front.
- Confirm that native Win shortcuts on ④ still work: Win+E, Win+V, Win+R, and tapping ④ for Start.

If the layout gets confused:

```powershell
lwm release-all-windows
```

Or stop LeopardWM and exit AutoHotkey from their tray icons. LeopardWM can also be stopped from a terminal:

```powershell
lwm stop
```

Once the direct-connection and KVM tests both pass:

```powershell
lwm autostart enable
```

## Where it still is not niri

This gets me close, but it does not erase the platform boundary.

| niri behavior | LeopardWM/AutoHotkey equivalent or limit |
| --- | --- |
| Dynamically managed workspaces | Nine numbered workspaces per monitor |
| `Mod+H/L` continues to another monitor at an edge | Basic left/right focus only |
| Move a whole column to a workspace or monitor | The corresponding commands move the focused window |
| `Mod+,/.` consumes or expels | The mapped actions only consume; expel is a separate LeopardWM action |
| Width and height change by 10% | LeopardWM has no step resize, so it cycles presets |
| `Mod+Ctrl+F` expands to available width | Approximated with maximize-column |
| `Mod+Shift+V` switches floating/tiled focus | Left unbound |
| `Mod+Ctrl+C` centers visible columns | Left unbound |
| `Mod+Shift+Y/U/I/O` moves or reorders workspaces | Left unbound |
| `Mod+Tab` returns to the previous workspace | Left unbound |
| `Mod+S` opens niri's screenshot UI | Snipping Tool region capture via AutoHotkey |
| `Mod+Escape` controls shortcut inhibition | Approximated with LeopardWM pause |
| Noctalia launcher and session UI | AutoHotkey Run dialog, menu, and help window |
| Wayland compositor behavior | Windows still uses DWM, and application frameworks vary |

Elevated and protected windows can also behave differently when LeopardWM runs without matching privileges. AutoHotkey has similar integrity-level boundaries when interacting with elevated applications. Secure screens and operating-system shortcuts remain outside either tool's control. That is fine; I am not trying to make either program responsible for the Windows security model.

## Takeaways

The useful change was not finding a Windows application with exactly niri's feature list. It was separating two jobs that had been conflated into one modifier:

- **F24 on the thumb Win key** is the physical window-management modifier, in the same place my thumb already reached for niri.
- **④ is Left GUI**, providing a real Win/Super key and restoring Win+L and every other native shortcut on Windows.

The Kinesis Mod key and its firmware layer are untouched, so nothing else on the keyboard had to move.

LeopardWM handles the layout and workspaces. AutoHotkey handles the small set of operating-system and application actions that do not belong in a window manager. Neither tool is asked to imitate Noctalia or Windows itself.

Now the KVM button changes which computer is running the applications, not which set of habits my hands need. Windows is still Windows, and some differences remain. But moving between home and work is back to being one button click instead of one button click plus a mental context switch.
