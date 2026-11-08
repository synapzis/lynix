# Lynix Proton Wrapper

A standalone Bash launcher for a **user-supplied** `UI.exe` (Lynx). It runs the
app through GDK-Proton or GE-Proton and installs the Microsoft WebView2 and
Visual C++ runtimes into a private Proton prefix.

- No Steam installation is required.
- `UI.exe` is not included, copied, downloaded, or embedded by this project.
  The script asks you to select your own copy on first run.
- WebView2 and the VC++ runtime are downloaded and installed automatically.
- WebView2 is configured with the startup and cursor fixes it needs under Wine.
- If WebView2 fails with exit status `5` or `247`, the launcher moves the
  WebView2 profile into a backup folder and retries once automatically.

## Quick start

Run the script directly:

```sh
./lynix-proton.sh
```

Or pipe it straight from a URL:

```sh
curl -fsSL https://raw.githubusercontent.com/alloc-nullptr/lynix/refs/heads/main/lynix-proton.sh | bash
```

When piping, pass options after `bash -s --`:

```sh
curl -fsSL https://raw.githubusercontent.com/alloc-nullptr/lynix/refs/heads/main/lynix-proton.sh | bash -s -- --setup
curl -fsSL https://raw.githubusercontent.com/alloc-nullptr/lynix/refs/heads/main/lynix-proton.sh | bash -s -- --exe /path/to/UI.exe --proton /path/to/Proton
```

Prompts are read from the controlling terminal (`/dev/tty`), so the
`curl ... | bash` form stays interactive. In a graphical session the script can
use `zenity`, `kdialog`, or `yad` instead.

### Save the script first

A saved copy is easier to re-run and is required for the application-menu
shortcut:

```sh
curl -fsSL https://raw.githubusercontent.com/alloc-nullptr/lynix/refs/heads/main/lynix-proton.sh -o ~/.local/bin/lynix-proton.sh
chmod +x ~/.local/bin/lynix-proton.sh
~/.local/bin/lynix-proton.sh
```

If you create the shortcut from a piped run, also set `LYNIX_SCRIPT_URL` so the
script can keep a copy of itself:

```sh
curl -fsSL "$URL" | LYNIX_SCRIPT_URL="$URL" bash -s -- --install-desktop
```

## What it does

1. Asks for your `UI.exe` (or uses `--exe PATH`) and saves the choice.
2. Finds an installed GDK-Proton, GE-Proton, or Steam Proton build.
   If none exists, it offers to download one into its private data folder.
3. Creates a private Proton prefix that does not depend on Steam.
4. Installs the x64 Visual C++ runtime and the WebView2 runtime into that
   prefix if they are missing.
5. Configures the WebView2 machine policy for `UI.exe` and for Roblox's
   WebView2 instances, plus the Wine X11 cursor settings.
6. Launches `UI.exe` through Proton.
7. If the app exits with status `5` or `247`, moves the `UI.exe` WebView2
   profile into the backup folder and retries once.

The first setup downloads roughly 600 MB and needs about 3 GB of free disk
space. Later runs use the saved configuration and skip the downloads.

## Options

```
-h, --help                 Show help and exit
--select, --change-exe     Select UI.exe again (file picker)
--exe PATH                 Use PATH as UI.exe
--proton PATH              Use the Proton build in PATH
--download-proton FLAVOR   Download GDK-Proton (gdk) or GE-Proton (ge)
--setup                    Install/configure everything, do not launch
--status                   Show the current configuration
--install-desktop          Create an application-menu shortcut
--uninstall                Remove the shortcut and wrapper settings
--purge                    With --uninstall, also delete private data
--no-gui                   Never use zenity/kdialog; prompt in the terminal
--debug                    Keep verbose Proton/Wine output and WebView2 logs
--reset                    Forget saved choices, then run
--reset-webview            Move all WebView2 profiles (UI.exe and Roblox)
                           to the backup folder before launching
--version                  Print the version
```

Pass arguments to `UI.exe` after `--`:

```sh
./lynix-proton.sh -- --some-flag
```

## WebView2 and cursor fixes

WebView2 has a few Wine-specific problems. The launcher writes a machine policy
inside the private prefix, so the fixes apply to the WebView2 instances hosted
by both `UI.exe` and Roblox.

The policy value is:

```
--disable-features=HideCursorWhileTyping --disable-gpu
```

- `--disable-gpu` avoids a WebView2 GPU-process startup failure under Wine
  that makes `UI.exe` exit with status `5`.
- `--disable-features=HideCursorWhileTyping` works around a WebView2 152-156
  regression that could hide the pointer or leave it stuck inside the app.
- Keep `--disable-gpu` **last**. Chromium parses the policy string as a
  command line; a switch at the beginning can be discarded.
- The policy is stored under
  `HKLM\SOFTWARE\Policies\Microsoft\Edge\WebView2\AdditionalBrowserArguments`
  in the private prefix. Specific executable names override `"*"`, so every
  entry contains the full argument set.

WebView2 logs are only enabled with `--debug`. Chromium's `--enable-logging`
always creates a visible console window for `msedgewebview2.exe`; this is
by design on Windows and cannot be suppressed while logging is on.

The launcher also derives `XCURSOR_THEME` and `XCURSOR_SIZE` from the desktop
settings and writes these Wine X11 options into the private prefix:

```
GrabFullscreen=N
UseSystemCursors=Y
```

This keeps the pointer from being confined to the app and makes Wine use the
same cursor theme and size as the rest of the desktop instead of a built-in
fallback cursor.

The app runs on Xwayland by default. Proton's native Wayland driver can make
WebView2 content render as a black window on some systems. Set
`LYNIX_ENABLE_WAYLAND=1` only if you want to experiment with native Wayland.

## Data locations

| What | Default path |
| --- | --- |
| Settings | `$XDG_CONFIG_HOME/lynix-proton/config` |
| Prefix, Proton, cache, logs | `$XDG_DATA_HOME/lynix-proton` |
| Launcher log | `$XDG_DATA_HOME/lynix-proton/logs/launcher.log` |
| WebView2 profile backups | `$XDG_DATA_HOME/lynix-proton/backups` |
| `UI.exe` WebView2 profile | `.../prefix/pfx/drive_c/users/steamuser/AppData/Local/com.savage.lynx/EBWebView` |
| Roblox WebView2 profiles | `.../AppData/Local/Roblox/**/EBWebView` |

Profiles are moved to the backup folder, never deleted. To restore one, close
the app and move the backup folder back to its original location.

## Environment variables

| Variable | Meaning |
| --- | --- |
| `LYNIX_DATA_DIR` | Override the private data folder |
| `LYNIX_CONFIG_DIR` | Override the settings folder |
| `LYNIX_PROTON` | Use this Proton folder without saving it |
| `LYNIX_SCRIPT_URL` | Script URL used by `--install-desktop` when run from a pipe |
| `LYNIX_ENABLE_WAYLAND` | Set to `1` to use Proton's native Wayland driver |
| `LYNIX_NO_WEBVIEW_RETRY` | Set to `1` to disable the automatic WebView2 recovery |

## Troubleshooting

Show the current setup:

```sh
./lynix-proton.sh --status
```

Read the launcher log:

```sh
tail -n 100 ~/.local/share/lynix-proton/logs/launcher.log
```

### Exit status 5 or 247

The launcher handles these automatically: it moves the `UI.exe` WebView2
profile to the backup folder and launches one more time with a fresh profile.
The old profile (cookies, settings, and cache) remains in the backup folder.

If the retry also fails:

```sh
./lynix-proton.sh --reset-webview
./lynix-proton.sh
```

`--reset-webview` backs up the `UI.exe` profile and every Roblox WebView2
profile. Then check that the WebView2 runtime is current:

```sh
./lynix-proton.sh --status
```

The first start after a force-kill or a profile reset can take 30-60 seconds
while the app shows "Login / Checking saved credentials…". A first attempt may
also fail once after a previous session; the automatic retry covers that case.
To disable the retry and keep the profile untouched, set
`LYNIX_NO_WEBVIEW_RETRY=1`.

### Hidden, stuck, or wrong cursor

Run `--setup` once so the current version writes the WebView2 policy and Wine
cursor settings:

```sh
./lynix-proton.sh --setup
```

Close the app completely, then start it again. The app uses the same cursor
theme and size as the KDE/GNOME desktop. If you change the desktop cursor
theme or size, close the app and relaunch it so the new values are exported.

### A console window appears

A console window is expected only with `--debug`, because WebView2 logging is
enabled in that mode. Run without `--debug` for normal use.

### WebView2 content is black

Make sure native Wayland is not enabled:

```sh
unset LYNIX_ENABLE_WAYLAND
./lynix-proton.sh
```

The default Xwayland path is the tested configuration.

### Stale app IPC files

If the app was force-killed and a later start exits immediately, remove the
stale IPC files and try again:

```sh
rm -f ~/.local/share/lynix-proton/prefix/pfx/drive_c/Lynx/mcp/endpoint.json
rm -f ~/.local/share/lynix-proton/prefix/pfx/drive_c/Lynx/port
```

The wrapper does this automatically when it starts a new instance.

## Requirements

- Linux with a working Vulkan driver (AMD, Intel, or NVIDIA)
- Bash 4.3 or newer
- `curl` or `wget`, plus `tar`
- Optional: `zenity`, `kdialog`, or `yad` for graphical prompts

## Legal

`UI.exe` is licensed software and belongs to its author. This project only
provides a launcher; it does not distribute the application.

big thanks to deepseek
