# Pop!_OS / Linux Custom System Configuration & Optimizations

This repository contains details and scripts for custom system optimizations applied to an **MSI laptop** featuring an **Intel Core Raptor Lake-P CPU** and **Intel Iris Xe Graphics**, running the Wayland-based **COSMIC Desktop** on Pop!_OS.

---

## 💻 1. Graphics Driver & Display Stability (Intel Xe)
By default, older/legacy Intel drivers use `i915`. We switched to the modern `xe` driver. However, on Raptor Lake-P, `xe` is experimental and has power-saving synchronization bugs (like Panel Self Refresh) which trigger display controller crashes and black screens.

### Blacklist `i915`
Create `/etc/modprobe.d/blacklist-i915.conf` to block the legacy driver:
```text
blacklist i915
```

### Force Enable `xe` with PSR Disabled
Create `/etc/modprobe.d/xe.conf` to force-probe the GPU and disable Panel Self Refresh (PSR) to prevent plane/CRTC crashes:
```text
options xe force_probe=a7a1 enable_psr=0
```

---

## 🔋 2. PowerTOP GPU Suspend Override
PowerTOP's `--auto-tune` service forces the graphics card's PCI power state to `auto`. Waking up the Intel GPU display engine from runtime suspend under the experimental `xe` driver causes display engine hangs. 

We keep PowerTOP running for other hardware (Wi-Fi, USB, Audio, etc.) but override it for the GPU.

Update `/etc/systemd/system/powertop.service`:
```ini
[Unit]
Description=PowerTOP auto-tune
Documentation=man:powertop(1)

[Service]
Type=oneshot
ExecStart=/usr/sbin/powertop --auto-tune
ExecStartPost=/bin/sh -c "echo on > /sys/bus/pci/devices/0000:00:02.0/power/control"

[Install]
WantedBy=multi-user.target
```
*Run `sudo systemctl daemon-reload` after updating.*

---

## ⚡ 3. Persistent Power Profile (Battery Mode & Quiet Fans)
Pop!_OS defaults to `Balanced` mode on boot. On Raptor Lake-P, Intel Turbo Boost causes aggressive fan spikes. Locking the system to `battery` disables Turbo Boost and caps the CPU frequency range to `10% - 50%`, keeping the CPU cool and silent.

Create `/etc/systemd/system/persist-power-profile.service`:
```ini
[Unit]
Description=Persist system76-power battery profile on boot
After=system76-power.service
Wants=system76-power.service

[Service]
Type=oneshot
ExecStart=/usr/bin/system76-power profile battery
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now persist-power-profile.service
```

---

## 📂 4. Indexing & File-Watching Exclusions (Home Directory)
Since our workspace root is our home folder (`/home/tankisompela`), AI tools (`agy`, `opencode`) scan and watch the entire directory. Caches, configuration folders, downloads, and package directories cause massive CPU spikes and lag.

Create `~/.ignore` and `~/.gitignore` files in the home directory (`~`):
```text
# System & Cache Directories
.cache/
.config/
.local/
.npm/
.cargo/
.rustup/
.gemini/
.mozilla/
Downloads/
.opencode/

# Development & Build Artifacts
**/node_modules/
**/.git/
**/.venv/
**/dist/
**/build/
**/.next/
**/.gradio/
```

### `opencode` Global Configuration
Update `~/.config/opencode/opencode.json` to expand the watcher ignore list:
```json
  "watcher": {
    "ignore": [
      "**/node_modules/**",
      "**/.git/**",
      "**/.venv/**",
      "**/dist/**",
      "**/build/**",
      "**/.next/**",
      "**/.gradio/**",
      "**/.cache/**",
      "**/.config/**",
      "**/.local/**",
      "**/Downloads/**",
      "**/.npm/**",
      "**/.mozilla/**",
      "**/.opencode/**",
      "**/.gemini/**",
      "**/.rustup/**",
      "**/.cargo/**"
    ]
  }
```

---

## 🛡️ 5. Resource Limits & Home Directory Blocks for AI CLIs
We wrap AI CLI tools (`agy`, `opencode`, `kilo`, and `qwen`) in Zsh functions to:
1. Prevent them from running in the home directory (displaying a warning and confirmation prompt).
2. Cap their CPU, RAM, and disk priorities using a `systemd` user scope.

Add to `~/.zshrc`:
```zsh
# Helper function to prevent running AI tools in the home directory
_check_home_dir() {
    if [ "$PWD" = "$HOME" ]; then
        echo -e "\033[1;33m⚠️  WARNING: You are running an AI tool inside your home directory ($HOME)!\033[0m"
        echo -e "This causes high CPU usage, system lag, and fan noise due to file indexing."
        echo -e "It is recommended to 'cd' into a specific project folder first (e.g., ~/Projects/your-project)."
        echo -n "Are you sure you want to run this in your home directory? (y/N): "
        read -r response
        if [[ "$response" != [yY] ]]; then
            echo "Operation cancelled. Please cd to your project directory."
            return 1
        fi
    fi
    return 0
}

# Wrapper functions applying systemd limits (800MB RAM cap, minimal CPU shares)
opencode() {
    _check_home_dir || return 1
    if command -v systemd-run >/dev/null 2>&1; then
        systemd-run --user --scope --quiet -p CPUWeight=1 -p MemoryMax=800M -p IOWeight=1 /home/tankisompela/.opencode/bin/opencode "$@"
    else
        nice -n 19 ionice -c 3 /home/tankisompela/.opencode/bin/opencode "$@"
    fi
}

agy() {
    _check_home_dir || return 1
    if command -v systemd-run >/dev/null 2>&1; then
        systemd-run --user --scope --quiet -p CPUWeight=1 -p MemoryMax=800M -p IOWeight=1 /home/tankisompela/.local/bin/agy "$@"
    else
        nice -n 19 ionice -c 3 /home/tankisompela/.local/bin/agy "$@"
    fi
}

kilo() {
    _check_home_dir || return 1
    if command -v systemd-run >/dev/null 2>&1; then
        systemd-run --user --scope --quiet -p CPUWeight=1 -p MemoryMax=800M -p IOWeight=1 /home/tankisompela/.kilo/bin/kilo "$@"
    else
        nice -n 19 ionice -c 3 /home/tankisompela/.kilo/bin/kilo "$@"
    fi
}

qwen() {
    _check_home_dir || return 1
    if command -v systemd-run >/dev/null 2>&1; then
        systemd-run --user --scope --quiet -p CPUWeight=1 -p MemoryMax=800M -p IOWeight=1 /home/tankisompela/.nvm/versions/node/v24.17.0/bin/qwen "$@"
    else
        nice -n 19 ionice -c 3 /home/tankisompela/.nvm/versions/node/v24.17.0/bin/qwen "$@"
    fi
}
```

---

## 🔒 6. Supply-Chain Security (Malware Protection for npm/npx)
We installed `npq` and `socket` globally under your active NVM Node version (`v24.17.0`) to check for supply-chain attacks.

### Automatic Scanning (Socket Wrapper)
Add aliases to `~/.zshrc` (automatically configured by running `socket wrapper on`):
```zsh
alias npm="socket npm"
alias npx="socket npx"
```
Whenever you run `npm install <package>`, Socket runs static analysis scans to block unexpected filesystem/network calls, obfuscated code, or typosquatting threats. No token is needed.

### Pre-install Audits (npq Gatekeeper)
Add to `~/.zshrc` to require manual prompts and disable auto-continue counts for `npq install <package>` audits:
```zsh
export NPQ_DISABLE_AUTO_CONTINUE=true
```

---

## 🌐 7. Google Chrome Hardware Acceleration (Wayland & Vulkan)
To enable hardware video decoding (VA-API) and Vulkan compositing on your Intel GPU under Wayland (minimizing CPU utilization and fan noise during video playback), open `chrome://flags` and configure:

* **Override software rendering list (`#ignore-gpu-blocklist`)** -> `Enabled`
* **Zero-copy rasterizer (`#enable-zero-copy`)** -> `Enabled`
* **Vulkan (`#enable-vulkan`)** -> `Enabled`
* **Default ANGLE Vulkan (`#default-angle-vulkan`)** -> `Enabled`
* **Vulkan from ANGLE (`#vulkan-from-angle`)** -> `Enabled`
* **Force enable WebGPU interop (`#force-enable-webgpu-interop`)** -> `Enabled`
