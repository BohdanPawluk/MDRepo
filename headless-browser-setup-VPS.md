# Headless Browser Setup on a VPS for Hermes

Documented: October 5, 2026

## Purpose

Set up Chromium as a systemd-managed headless browser that Hermes
can control locally and through its already-configured Telegram bot.

This setup does not use Docker and does not depend on Hermes
automatically launching Chromium.

## Configuration used

- Hermes configuration: `~/.hermes/config.yaml`
- Browser service: `~/.config/systemd/user/chromium-hermes.service`
- Chrome DevTools Protocol (CDP) endpoint: `http://127.0.0.1:9222`
- Dedicated browser profile: `~/.config/hermes-browser`
- Service owner: the Linux user running Hermes

Important naming corrections:

- The configuration filename used here is `config.yaml`, not `config.yml`.
- The service filename is lowercase `chromium-hermes.service`.
- Linux filenames are case-sensitive.

## 1. Check prerequisites and record the environment

Log in as the ordinary Linux user that runs Hermes.

```bash
whoami
cat /etc/os-release
uname -m
systemctl --user --version
command -v hermes
```

Requirements:

- A Linux VPS with systemd and a working user service manager.
- Sudo access for package installation and enabling lingering.
- Hermes already installed with browser tools available.
- Outbound internet access.
- Telegram integration already configured if Telegram testing is required.

This document covers the browser service and connection configuration.
It does not replace the Hermes installation or Telegram setup guide.

Do not run Chromium as root or disable its sandbox as a default workaround.

## 2. Install utilities, Chromium, and its runtime libraries

Choose the installation method matching the VPS operating system.
Do not run both installation branches.

If Chromium is already installed and working, keep that installation
and proceed to checking its executable path.

### Debian: native Chromium package

```bash
sudo apt update
sudo apt install -y chromium chromium-sandbox ca-certificates curl nano
```

APT downloads Chromium and installs its required shared libraries.

The Chromium dependency list includes libraries for:

- NSS and NSPR: TLS and browser security.
- GLib, GTK, ATK, and AT-SPI: platform and accessibility support.
- Cairo, Pango, and HarfBuzz: rendering and text.
- X11, XCB, and related libraries: Chromium platform support,
  even when running headlessly.
- ALSA/PulseAudio: audio support.
- Image, compression, and media codecs.

Examples of package names on Debian 12 include:

- `libnss3`
- `libnspr4`
- `libglib2.0-0`
- `libgtk-3-0`
- `libatk1.0-0`
- `libatk-bridge2.0-0`
- `libcairo2`
- `libpango-1.0-0`
- `libasound2`
- `libx11-6`
- `libxcb1`
- `libxcomposite1`
- `libxdamage1`
- `libxrandr2`
- `libxkbcommon0`

This is an explanatory list, not a universal manual installation command.
Package names and dependencies vary by OS release. Let APT resolve them.

### Ubuntu: Chromium Snap package

On supported Ubuntu releases, `chromium-browser` is a transitional
package for the Chromium Snap.

```bash
sudo apt update
sudo apt install -y snapd ca-certificates curl nano
sudo snap install chromium
```

Snap supplies Chromium and its packaged runtime dependencies.

If Snap was newly installed, follow any logout or reboot instructions
before continuing.

For Snap Chromium, use a profile directory under its writable
application-data location:

```bash
mkdir -p "$HOME/snap/chromium/common/hermes-browser"
```

In the service below, replace the profile argument with:

```text
--user-data-dir=%h/snap/chromium/common/hermes-browser
```

### Hermes browser-tool dependencies

The working machine already has the dependencies needed for its
installed Hermes browser backend.

For future client releases, record and reproduce the tested Hermes
revision and its official installation requirements. Do not guess a
new Python or Node dependency list based on a different Hermes version.

## 3. Find and verify the Chromium executable

```bash
command -v chromium-browser
command -v chromium
```

Typical paths include:

- `/usr/bin/chromium-browser`
- `/usr/bin/chromium`
- `/snap/bin/chromium`

Use the actual working path in the systemd service's `ExecStart`.

Check its version using that path. For example:

```bash
/usr/bin/chromium-browser --version
```

If that path does not exist, use the path returned by `command -v`.

## 4. Configure Hermes

Back up the existing configuration before editing:

```bash
cp -a ~/.hermes/config.yaml \
  ~/.hermes/config.yaml.backup-$(date +%Y%m%d-%H%M%S)

nano ~/.hermes/config.yaml
```

Add or update this configuration:

```yaml
browser:
  cdp_url: http://127.0.0.1:9222
```

Formatting rules:

- `browser:` starts at the beginning of the line.
- `cdp_url:` is indented by two spaces.
- Use spaces, not tabs.
- Leave one space after the colon before the URL.
- If a `browser:` section already exists, add `cdp_url` inside it.
- Do not create a second top-level `browser:` section.
- Preserve existing unrelated settings.

Save in nano:

1. Press `Ctrl+O`.
2. Press `Enter`.
3. Press `Ctrl+X`.

If Hermes runs with a custom `HERMES_HOME` or a separate configuration
profile, edit the configuration used by that actual process.

## 5. Create the systemd user service

Check the existing directory:

```bash
ls -la ~/.config/systemd/user/
```

Create it if necessary:

```bash
mkdir -p ~/.config/systemd/user
```

For a native Chromium installation, create the dedicated profile directory:

```bash
mkdir -p ~/.config/hermes-browser
chmod 700 ~/.config/hermes-browser
```

For Snap Chromium, use the profile directory from Step 2 instead.

Open the service file:

```bash
nano ~/.config/systemd/user/chromium-hermes.service
```

Paste:

```ini
[Unit]
Description=Chromium for Hermes browser automation

[Service]
Type=simple
ExecStart=/usr/bin/chromium-browser --headless --remote-debugging-address=127.0.0.1 --remote-debugging-port=9222 --user-data-dir=%h/.config/hermes-browser --no-first-run --no-default-browser-check about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Before saving:

- Replace `/usr/bin/chromium-browser` with the executable verified in Step 3.
- If using Snap, replace the profile argument as described in Step 2.
- Keep `ExecStart` on one line.
- `%h` represents the service user's home directory.

This starts Chromium without a visible browser window and enables its
local debugging endpoint.

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

## 6. Stop any manually launched Chromium instance

If Chromium is still running in a foreground terminal from testing,
press `Ctrl+C` in that terminal before starting the service.

Do not kill every Chromium process indiscriminately: other browser
sessions may be unrelated.

Check whether port 9222 is already occupied:

```bash
ss -ltnp 'sport = :9222'
```

Only one intended browser instance should own this port.

## 7. Enable operation after logout and at boot

User services need lingering to run without an active login session.

Run as the intended Hermes user:

```bash
sudo loginctl enable-linger "$(whoami)"
loginctl show-user "$(whoami)" -p Linger
```

Expected output:

```text
Linger=yes
```

This keeps the user service manager available after logout and starts
it at boot.

## 8. Reload, enable, and start the service

Run these commands as the Hermes user, without `sudo`:

```bash
systemctl --user daemon-reload
systemctl --user enable --now chromium-hermes.service
systemctl --user status chromium-hermes.service --no-pager
```

Verify:

```bash
systemctl --user is-enabled chromium-hermes.service
systemctl --user is-active chromium-hermes.service
```

Expected results:

```text
enabled
active
```

- `daemon-reload` loads new or edited service definitions.
- `enable` enables startup through the user's default target.
- `--now` also starts the service immediately.

## 9. Verify the CDP endpoint

```bash
curl --fail --silent --show-error \
  http://127.0.0.1:9222/json/version
```

Expected result: JSON containing browser information and a
`webSocketDebuggerUrl`.

Check the listening address:

```bash
ss -ltnp 'sport = :9222'
```

The endpoint should be loopback-only, not publicly exposed.

Do not open port 9222 in the VPS firewall or cloud security group.
Treat browser remote debugging as privileged access to browser sessions.

## 10. Test Hermes and Telegram

If Hermes or the Telegram gateway was already running when the
configuration changed, restart the relevant process using its existing
service name and normal restart procedure.

Do not invent a new Hermes service name. Check existing units if needed:

```bash
systemctl --user list-units --type=service --all
```

Send this message in Hermes CLI or to the configured Telegram bot:

> Use browser_exec to open https://example.com and report the page title.

Expected title:

> Example Domain

The earlier Telegram test returned "🐴 Example Domain"; the emoji was
part of the bot response, not the page title itself.

For a stronger check, also ask it to report the final URL.

In Telegram, send a normal request to use the browser tool.
Do not rely on the interactive CLI `/browser connect` command there.

## 11. Verify restart and reboot behavior

Restart the browser service:

```bash
systemctl --user restart chromium-hermes.service
```

Repeat the CDP and Telegram tests.

When safe to interrupt the VPS, reboot:

```bash
sudo reboot
```

After reconnecting:

```bash
systemctl --user is-active chromium-hermes.service
curl --fail --silent --show-error \
  http://127.0.0.1:9222/json/version
```

Repeat the Telegram test.

Browser startup alone does not start Hermes, its Telegram gateway,
or a task scheduler. Their existing services must also be configured
for unattended operation.

## 12. Troubleshooting and maintenance

### View logs

```bash
journalctl --user -u chromium-hermes.service -n 100 --no-pager
```

### Executable not found

Repeat Step 3 and correct `ExecStart`.

Then reload and restart:

```bash
systemctl --user daemon-reload
systemctl --user restart chromium-hermes.service
```

### Port conflict or profile already in use

Stop the old manual instance using the same port or profile.
Do not delete the profile to solve this without understanding the issue.

### Missing shared library

Record the exact error, confirm the OS release, and install or repair
the appropriate distribution package.

Do not copy a library-install command from a different OS release.

### Snap profile permission error

Use the Snap-specific profile directory from Step 2.

### Service fails repeatedly

Read the logs and fix the underlying problem first. Then:

```bash
systemctl --user reset-failed chromium-hermes.service
systemctl --user start chromium-hermes.service
```

### Browser works locally but not in Telegram

Check that the Telegram gateway:

- Runs as the intended Linux user.
- Loads the configuration edited in Step 4.
- Has browser tools enabled.
- Was restarted after configuration changes, if required.

### Stop the browser

```bash
systemctl --user stop chromium-hermes.service
```

### Disable automatic startup

```bash
systemctl --user disable --now chromium-hermes.service
```

## 13. Client-release checklist

- [ ] Record the tested Linux distribution, release, and architecture.
- [ ] Record the tested Hermes revision.
- [ ] Record the Chromium package source and version.
- [ ] Install Chromium and its distribution-managed dependencies.
- [ ] Verify the executable path.
- [ ] Configure `browser.cdp_url` in the correct `config.yaml`.
- [ ] Create the correct dedicated browser profile directory.
- [ ] Install `chromium-hermes.service`.
- [ ] Enable lingering for the intended service user.
- [ ] Enable and start the browser service.
- [ ] Verify the local CDP endpoint.
- [ ] Verify Hermes CLI browser operation.
- [ ] Verify Telegram browser operation.
- [ ] Verify operation after reboot and logout.
- [ ] Keep port 9222 private.
- [ ] Use separate credentials and browser profiles for each client.
- [ ] Never ship your personal browser profile or session cookies.
- [ ] Test scheduled workflows separately, including duplicate-run prevention.
- [ ] Apply security updates and retest the workflow after updates.

## References

- Hermes browser configuration:
  https://hermes-agent.nousresearch.com/docs/user-guide/features/browser/
- Debian Chromium package and dependencies:
  https://packages.debian.org/bookworm/chromium
- Ubuntu Chromium packaging:
  https://packages.ubuntu.com/en/chromium-browser
- Ubuntu Chromium installation:
  https://snapcraft.io/install/chromium/ubuntu
- systemd user lingering:
  https://www.freedesktop.org/software/systemd/man/252/loginctl.html
- Chromium headless operation:
  https://chromium.googlesource.com/chromium/src/+/refs/tags/122.0.6261.171/headless/
