# BLACKARCH_SETUP

Post-installation recovery guide and automated fix script for BlackArch Linux after a Calamares installer deployment. Resolves the broken database sync loop, PGP "Unknown Trust" errors, Reflector path failures, and Python version mismatch that ship with the 2023+ Slim ISO.

---

## First Things First: Fix the Calamares Installer

The BlackArch Slim ISO ships a Calamares configuration that attempts to synchronize package databases during installation. Because the ISO's signing keys and mirrors are stale, this sync fails in a loop and the installer never completes. **You must patch one config file before launching the installer.**

### Steps (pre-install, from the live session)

1. **Boot the ISO** — default credentials are `root` / `blackarch`.

2. **Open a root file manager** — the config file is owned by root:

   ```bash
   sudo thunar
   ```

3. **Navigate to the Calamares modules directory:**

   ```
   /etc/calamares/modules/
   ```

4. **Open `packages.conf`** and find the `update_db` parameter.

5. **Change it from `true` to `false`:**

   ```yaml
   update_db: false
   ```

6. **Save the file**, close the editor, then launch **Install BlackArch** from the desktop.

The installer will now skip the broken database sync and complete normally. All repository and keyring issues are handled post-install by the script below.

---

## Post-Install: Automated Recovery

After the base system is installed and you've booted into it, clone this repo and run the fix script. It handles everything in sequence: keyring wipe/rebuild, BlackArch key injection, mirror fallback, and full system upgrade.

### Quick Start

```bash
git clone https://github.com/0xb0rn3/BLACKARCH_SETUP.git
cd BLACKARCH_SETUP
chmod +x fix-blackarch.sh
sudo ./fix-blackarch.sh
```

### What the Script Does

| Phase | Action | Why |
|-------|--------|-----|
| 1 | Kill stale `gpg-agent` processes | Prevents keyring init deadlocks |
| 2 | Wipe `/etc/pacman.d/gnupg` and rebuild from scratch | The ISO ships a corrupted/expired keyring |
| 3 | `--populate archlinux` + `--populate blackarch` | Restores all upstream signing keys |
| 4 | Receive and locally sign BlackArch dev key (`F9A6E68A...`) | Resolves "Unknown Trust" on every BlackArch package |
| 5 | Write manual mirrorlist (kernel.org, pkgbuild.com, rackspace) | Bypasses Reflector, which fails on a stale ISO |
| 6 | Disable `reflector.service` and `reflector.timer` | Prevents Reflector from overwriting the fixed mirrorlist |
| 7 | Verify `[blackarch]` repo block exists in `pacman.conf` | Adds it if missing, creates `blackarch-mirrorlist` |
| 8 | `pacman -Syyu --overwrite='*'` | Full system upgrade; `--overwrite` handles file conflicts from major version jumps (e.g., Python 3.10 → 3.14) |
| 9 | Resolve JDK/JRE conflict, set default via `archlinux-java` | `jdk-openjdk` now provides `jre-openjdk`; both cannot coexist. Installs JDK if missing, removes standalone JRE if conflicting |
| 10 | Remove orphaned packages, trim package cache | Cleanup |

### Script Options

```
Usage: sudo ./fix-blackarch.sh [OPTIONS]

Options:
  -h, --help           Show help and exit
  -v, --verbose        Enable debug output
  -s, --skip-upgrade   Fix keyring/mirrors only, skip the full -Syyu
  -m, --mirror URL     Override default Arch mirror
  --no-color           Disable colored output

Examples:
  sudo ./fix-blackarch.sh                        # Full fix + upgrade
  sudo ./fix-blackarch.sh -v                     # Verbose mode
  sudo ./fix-blackarch.sh -s                     # Keyring fix only
  sudo ./fix-blackarch.sh -m "https://my.mirror/archlinux/\$repo/os/\$arch"
```

Logs are written to `/var/log/blackarch-fix-<timestamp>.log`.

---

## Manual Recovery (If You Prefer Doing It by Hand)

### 1. Wipe and Rebuild the Keyring

```bash
sudo rm -rf /etc/pacman.d/gnupg
sudo pacman-key --init
sudo pacman-key --populate archlinux
```

### 2. Inject the BlackArch Signing Key

```bash
sudo pacman-key --recv-keys F9A6E68A711354D84A9B91637533BAFE69A25079 --keyserver hkps://keyserver.ubuntu.com
sudo pacman-key --lsign-key F9A6E68A711354D84A9B91637533BAFE69A25079
```

If `keyserver.ubuntu.com` times out, try `keys.openpgp.org` or `pgp.mit.edu`.

### 3. Set a Working Mirrorlist

```bash
echo "Server = https://mirrors.kernel.org/archlinux/\$repo/os/\$arch" | sudo tee /etc/pacman.d/mirrorlist
```

### 4. Upgrade Everything

```bash
sudo pacman -Syyu --overwrite='*'
```

The `--overwrite='*'` flag is required because the Python major version jump (3.10 → 3.14) causes file conflicts in `/usr/lib/python3.*/site-packages/`. Without it, pacman will refuse to proceed.

### 5. Fix Java JDK/JRE Conflict

```bash
# Remove standalone JRE if JDK is also installed
sudo pacman -Rdd jre-openjdk 2>/dev/null

# Ensure JDK is installed (provides JRE)
sudo pacman -S jdk-openjdk --needed

# Set the active environment via archlinux-java (not manual JAVA_HOME exports)
sudo archlinux-java set java-$(java --version 2>/dev/null | grep -oP '\d+' | head -1)-openjdk
```

### 6. Reboot

```bash
sudo reboot
```

---

## Known Issues

**Reflector overwrites mirrorlist on boot** — The ISO enables `reflector.service` by default. If your mirrors break again after reboot, disable it:

```bash
sudo systemctl disable --now reflector.service reflector.timer
```

**Python site-packages mismatch** — If tools fail with `ModuleNotFoundError` after upgrade, the old 3.10 site-packages directory is stale. Reinstall affected packages:

```bash
sudo pacman -S python-<package> --overwrite='*'
```

**BlackArch repo not in pacman.conf** — Some ISO variants don't write the `[blackarch]` block. The script adds it automatically. To verify:

```bash
grep "\[blackarch\]" /etc/pacman.conf
```

**Java JDK/JRE conflict** — `jdk-openjdk` now provides `jre-openjdk`. If both are installed, pacman will error on upgrades. The script resolves this automatically. To fix manually:

```bash
# Remove standalone JRE (JDK provides it)
sudo pacman -Rdd jre-openjdk

# Install JDK if not present
sudo pacman -S jdk-openjdk

# Set default environment (never manually export JAVA_HOME with multiple versions)
sudo archlinux-java set java-<version>-openjdk

# Verify
archlinux-java status
```

**Java CVE Advisory (Q1 2026)** — Two critical vulnerabilities affect Java environments:

| CVE | Description | CVSS |
|-----|-------------|------|
| CVE-2026-29000 | pac4j Authentication Bypass | 10.0 |
| CVE-2025-27363 | Oracle Java SE RCE (Active Exploitation) | High |

Ensure your JDK is at the latest patched version after running the upgrade.

---

## Reference

- [Looking at BlackArch Linux in 2026 (Video)](https://www.youtube.com/watch?v=...) — Source for the Calamares `update_db` workaround
- [BlackArch Installation Guide](https://blackarch.org/downloads.html)
- [Arch Wiki: Pacman/Package Signing](https://wiki.archlinux.org/title/Pacman/Package_signing)

---

## License

MIT

## Author

**0xb0rn3** — [github.com/0xb0rn3](https://github.com/0xb0rn3) · [oxbv1@proton.me](mailto:oxbv1@proton.me)
