---
name: ubuntu-remove-snap-firefox-deb
description: Fully uninstall snap (including snapd) from Ubuntu and install deb-based Firefox from Mozilla's official APT repo. Use when the user asks to "remove snap / completely delete snapd / install Firefox without snap / switch Firefox to the deb version".
---

# Ubuntu: fully remove snap + install deb Firefox

Execute two phases in order: **remove snap first, then install Firefox**. Every `sudo` step needs the user's password — hand the commands to the user to run; do not pretend to run them yourself. Introduce no snap components at any point.

## Phase 1: remove snap

### Step 1: survey

```bash
snap list                          # installed snap packages
dpkg -l | grep -iE "firefox|snapd" # is firefox the snap transitional package (version like 1:1snap1-0ubuntu2)?
apt-cache rdepends --installed snapd
apt-get -s purge snapd             # dry run: confirm which packages purge actually removes
```

Completion criterion: you have the snap package list, and the simulation output shows **nothing** that would cascade into desktop components (gnome-shell, pulseaudio, etc.) before proceeding to Step 2.

Key fact: `ubuntu-desktop` / `ubuntu-desktop-minimal` only **Recommends** snapd — it is not a Depends. `apt-get -s purge snapd` should show exactly two removals: `snapd` and `firefox`. That is the green light to continue. If the simulation removes more, stop and analyze.

### Step 2: remove snap packages one by one (apps → runtimes → bases → snapd itself)

```bash
sudo snap remove --purge <app packages>       # firefox, snap-store, real apps
sudo snap remove --purge <runtime packages>   # gnome-XX-XXXX, gtk-common-themes, mesa-XXXX
sudo snap remove --purge bare core20 core22 core24
sudo snap remove --purge snapd
```

Substitute the actual package names; remove apps before bases. If a base refuses removal because it is in use, **skip it** — the next step forces it out.

### Step 3: remove system packages and leftovers

```bash
sudo apt purge -y snapd firefox              # firefox here is the snap transitional package
sudo apt autoremove --purge -y
sudo rm -rf /var/lib/snapd /var/snap /snap ~/snap
hash -r
```

If a squashfs mount point blocks deletion, reboot and rerun `rm -rf`.

### Step 4: verify

```bash
which snap snapd          # both: command not found
systemctl status snapd    # unit could not be found
ls -d /var/lib/snapd /var/snap /snap ~/snap   # none exist
mount | grep snap         # no squashfs mounts
```

All four passing means Phase 1 is done.

### Removal pitfalls (keep vs. delete)

| Package | Verdict | Reason |
|---|---|---|
| `libsnappy1v5` | **Keep** | Name resembles snap but it is a compression library; Chrome and others depend on it |
| `xdg-desktop-portal` | **Keep** | GTK file dialogs and desktop features need it; its description mentioning snap does not mean it depends on snapd |
| `libsnapd-glib1` / `gir1.2-snapd-1` | **Keep** | Removing them cascades into PulseAudio and GNOME Online Accounts; idle and harmless once snapd is gone |
| `ubuntu-desktop` metapackage | Keep (automatic) | Linked only via Recommends; purging snapd does not touch it |

## Phase 2: install Firefox from Mozilla's official APT repo (deb)

### Step 5: test repo reachability

```bash
curl -sI --max-time 8 https://packages.mozilla.org/apt/dists/mozilla/Release
```

Expect `HTTP/2 200`. If blocked, retry through the local proxy. If still unreachable, abort and report.

### Step 6: configure repo + double-pin

```bash
sudo install -d -m 0755 /etc/apt/keyrings
wget -q https://packages.mozilla.org/apt/repo-signing-key.gpg -O- | \
  sudo tee /etc/apt/keyrings/packages.mozilla.org.asc > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/packages.mozilla.org.asc] https://packages.mozilla.org/apt mozilla main" | \
  sudo tee /etc/apt/sources.list.d/mozilla.list

sudo tee /etc/apt/preferences.d/mozilla <<'EOF'
Package: *
Pin: origin packages.mozilla.org
Pin-Priority: 1000

Package: firefox
Pin: release o=Ubuntu*
Pin-Priority: -1
EOF
```

What the pin does, in two layers: priority 1000 on the Mozilla repo makes it always outrank Ubuntu's; Ubuntu's firefox is pinned to -1 (permanently refused), so the snap transitional package (`1:1snap1-*`) can never return.

### Step 7: install and verify

```bash
sudo apt update && sudo apt install -y firefox firefox-l10n-zh-cn
apt policy firefox
```

Completion criterion (from `apt policy firefox` output):
- Installed version looks like `157.0.1~build1` (the `~build1` suffix = Mozilla deb)
- Installed from `https://packages.mozilla.org/apt` at priority 1000
- Ubuntu's `1:1snap1-*` version marked `-1`

All three checks pass means the whole task is done. From here `apt upgrade` follows Mozilla's releases automatically, with no snapd involved at any point.
