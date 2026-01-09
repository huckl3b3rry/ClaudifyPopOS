# ClaudifyPopOS - AI Context

## AI Directives

### Software Installation Policy
**Always prefer native Linux versions over Windows emulation:**
- Choose Flatpak, native packages, or Linux reimplementations over Wine/Snap Wine packages
- Only use Windows emulation if there are serious reasons (specific features, compatibility requirements)
- Rationale: Better system integration, lower resource usage, better stability, proper Wayland/PipeWire support

## System Overview
- **OS**: Pop!_OS 24.04 LTS
- **Desktop**: Cosmic (with GTK fallback portals)
- **Audio Stack**: PipeWire 1.5.84 + WirePlumber + PulseAudio compatibility
- **GPU**: NVIDIA RTX 3050 (used for Ollama/RAG)
- **Kernel**: 6.17.9-76061709-generic

## Audio Devices
| Device | Type | Status |
|--------|------|--------|
| Sennheiser SP 20 for Lync | USB Speakerphone | Primary mic/speaker for calls |
| Galaxy Buds2 Pro | Bluetooth (AAC/A2DP) | Mobile audio, supports HSP/HFP for calls |
| Intel HDA PCH (ALC289) | Built-in | Backup |
| 4x HDMI outputs | Monitors | Display audio |

## Audio Configuration Files

### Udev Rules
- `/etc/udev/rules.d/81-bluetooth-power.rules` - Disables USB autosuspend for Intel AX211 Bluetooth
- `/etc/udev/rules.d/82-sennheiser-power.rules` - Disables USB autosuspend for Sennheiser SP 20

### BlueZ Configuration
- `/etc/bluetooth/main.conf` - **Keep minimal!** Experimental settings break Galaxy Buds connection
  - ControllerMode = dual
  - FastConnectable = false
  - AutoEnable = true

### PipeWire Configuration
- `~/.config/pipewire/pipewire.conf.d/10-audio-optimization.conf`
  - Quantum: 512 (10.7ms latency)
  - Allowed rates: 16000, 44100, 48000 Hz

### WirePlumber Rules
- `~/.config/wireplumber/wireplumber.conf.d/51-sennheiser-sp20.conf`
  - ALSA period-size: 1024, periods: 3, headroom: 2048
  - Priority: 2500, suspend timeout: 0
  - Larger buffers for USB stability
- `~/.config/wireplumber/wireplumber.conf.d/52-bluetooth-galaxy-buds.conf`
  - Node latency: 4800/48000 (100ms buffer)
  - Priority: 3000
  - Reconnect profiles enabled

### PipeWire Bluetooth Codecs (Compiled from source)
- `/usr/lib/x86_64-linux-gnu/spa-0.2/bluez5/` - AAC, LDAC, aptX, SBC codecs
- Original plugins backed up to `bluez5.backup/`

## Voice-to-Text Applications
| App | Model | Compute | Status |
|-----|-------|---------|--------|
| Voxtype | Built-in | CPU (AVX2) | Working (F12 hotkey) |
| SilentKeys | Parakeet (ONNX) | CPU | Working |
| Meetily | Parakeet (Ollama) | GPU | Working |

### Voxtype Service
- `~/.config/systemd/user/voxtype.service`
- Uses `sg input -c` wrapper for keyboard access
- Currently on voxtype-avx2 (Vulkan version crashes)

## Flatpak Overrides
- `~/.local/share/flatpak/overrides/com.github.IsmaelMartinez.teams_for_linux`
  - pulseaudio socket, pipewire-0 filesystem, all devices
  - Environment: PIPEWIRE_RUNTIME_DIR, PULSE_SERVER

## User Groups
- `audio`, `pipewire` - For RTKit priority
- `input` - For Voxtype keyboard access
- `docker`, `ollama` - For containers and AI

## Known Issues & Solutions

### Bluetooth Audio Dropouts (Galaxy Buds)
- **Cause**: Intel AX211 WiFi/BT combo chip coexistence + insufficient buffers
- **Fix**: Increased WirePlumber Bluetooth buffer to 100ms, udev rule for autosuspend
- **Warning**: Do NOT enable BlueZ Experimental settings - breaks connection!

### Teams Microphone Not Working
- **Cause**: USB autosuspend, missing groups, no WirePlumber rules
- **Fix**: Udev rule + groups + WirePlumber configs + Flatpak overrides

### USB Audio (Sennheiser) Dropouts
- **Cause**: Insufficient ALSA buffers, USB power management
- **Fix**: WirePlumber rules with larger period-size (1024) and headroom (2048)

### Voxtype F12 Not Working
- **Cause**: User not in input group for current session
- **Fix**: Modified systemd service to use `sg input -c` wrapper

### Voxtype Vulkan Crashes
- **Status**: Unresolved - using CPU (AVX2) version instead

### Galaxy Buds Won't Connect After BlueZ Changes
- **Cause**: Experimental=true or KernelExperimental=true in main.conf
- **Fix**: Revert to minimal BlueZ config, restart bluetooth service

## Pending/On Hold
- SilentKeys CUDA acceleration (waiting for RAG project to finish using GPU)
- Ollama+wtype script for GPU-accelerated voice-to-text

## Quick Commands
```bash
# Check audio status
wpctl status

# Restart audio stack
systemctl --user restart pipewire pipewire-pulse wireplumber

# Test microphone
pw-record /tmp/test.wav  # Ctrl+C to stop
pw-play /tmp/test.wav

# Check for audio errors
journalctl --user -u pipewire -u wireplumber --since "10 min ago" | grep -i error

# Connect Galaxy Buds
bluetoothctl connect F8:5B:6E:F5:F3:9D

# Set Bluetooth as default output
pactl set-default-sink $(pactl list sinks short | grep bluez | awk '{print $2}')

# Set Sennheiser as default
pactl set-default-sink alsa_output.usb-1395_Sennheiser_SP_20_for_Lync_A000420202700294-00.analog-stereo
pactl set-default-source alsa_input.usb-1395_Sennheiser_SP_20_for_Lync_A000420202700294-00.mono-fallback
```
