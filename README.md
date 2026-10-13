# ansible-role-ksuite

Infomaniak desktop apps on openSUSE Leap (x86_64), once per host for all users: kDrive (file sync), kChat and
kMeet. These are all Linux desktop apps Infomaniak offers (2026-10); Mail, Calendar, Euria, SwissTransfer etc.
are web/mobile only.

Per app listed in `apps`:

1. gets the download URL of the current version from Infomaniak (`https://www.infomaniak.com/<drive|kchat|kmeet>/latest`, JSON)
2. downloads the AppImage to `/opt/ksuite/<kDrive|kChat|kMeet>.AppImage` (root, 0755), checked against
   Infomaniak's checksum (kDrive: `<url>.sha256`, kChat/kMeet: sha512 in `latest-linux.yml`); it downloads again
   only when the checksum differs, i.e. on an update

The Linux entry in that JSON is looked up under several names (`release_keys` in `vars/main.yml`, Infomaniak
renames them); if none matches, the run stops with the names the JSON has now.

Also installs `fuse` (fusermount for the AppImage runtime). Apps not in `apps` are removed again (desired state);
`/opt/ksuite` stays, user data in the homes is not touched.

Only the executables: no menu entries, icons or autostart for all users (for now).

## Several users

The apps run per user without any configuration (unlike syncthing): kDrive's server process listens on a random
free localhost port and passes it to its GUI process; kChat/kMeet (Electron) have no listening port. The
"already running" check is per user, settings and data are in the home (`~/.config/kDrive`, `~/.config/kChat`,
`~/.config/kMeet`, `~/.local/state`). Credentials go into the user's GNOME keyring.

## Per user

- first start from a terminal or Alt+F2: `/opt/ksuite/kDrive.AppImage` (or `kChat`, `kMeet`), log in
  (kDrive: choose the folders), then switch on "launch at startup" in the app
- the apps' "launch at startup" writes `~/.config/autostart/…` with the path they were started from - the fixed
  paths in `/opt/ksuite` keep these entries valid across updates
- kDrive writes its own menu entry `~/.local/share/applications/kDrive.desktop` (no icon) at the login: it is the
  handler for the login redirect `kdrive://` - don't delete it
- kChat/kMeet get no menu entry and no `kchat://` / `kmeet://` handlers (links from the browser don't open the app)
- updates come with the next ansible run. kDrive only reports new versions on Linux; kChat switches its updater
  off because it can't write the AppImage; kMeet may still offer an update, installing it fails (the AppImage is
  root-owned) - ignore it. Running apps keep the old version until they are restarted.
- crash reports (Sentry) can't be switched off in kChat/kMeet; hosts: `sentry-desktop.infomaniak.com` (kDrive),
  `sentry-kchat.infomaniak.com` (kChat/kMeet)

kMeet always runs with `--no-sandbox` (built into the app). kChat uses Electron's sandbox (unprivileged user
namespaces, allowed on Leap); if it doesn't start: `/opt/ksuite/kChat.AppImage --no-sandbox`.

kDrive on Leap 16 with SELinux: its `libsentry.so` asks for an executable stack, which the default
`selinuxuser_execstack = off` refuses ("cannot enable executable stack ... Permission denied"). Start it with the
boolean on for the session only: `sudo setsebool selinuxuser_execstack on`, start kDrive, `sudo setsebool
selinuxuser_execstack off` (the running kDrive keeps working; after a reboot it's off anyway).

Tray icons in the GNOME top bar: extension "Status Icons" from package gnome-shell-classic (ansible-role-gnome).

## Variables

```yaml
ksuite:
  apps: [kdrive, kchat, kmeet]
```

All defaults: `defaults/main.yml`; the app list with URLs: `vars/main.yml`.

## Check

```bash
ls -l /opt/ksuite/
curl -s https://www.infomaniak.com/kchat/latest     # current version (also drive/latest, kmeet/latest)
ss -ltnp | grep -i kdrive                           # one random localhost port per running kDrive
```

## License

Apache-2.0

Created with the help of AI
