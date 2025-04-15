# AirPlay Audio Forwarding on Raspberry Pi (RAOP via PipeWire)

## Goal
Forward all system audio **from Raspberry Pi** to an **AirPlay-compatible speaker** (e.g., Edifier MS50A).

---

## Step 1: Install Required Packages

```bash
sudo apt update
sudo apt install -y pipewire wireplumber libpipewire-0.3-modules pulseaudio-utils
```

---

## Step 2: Set Up RAOP Module Loader in Script

Create `~/set-default-sink.sh`:

```bash
#!/bin/bash

TARGET="raop_sink.Cabinet-sound.local.192.168.1.194.7000"

# Load RAOP module if not active
if ! pactl list short modules | grep -q module-raop-discover; then
    pactl load-module module-raop-discover
fi

# Wait for sink to appear
for i in {{1..30}}; do
    if pactl list short sinks | grep -q "$TARGET"; then
        pactl set-default-sink "$TARGET"
        exit 0
    fi
    sleep 1
done

echo "Sink $TARGET not found after waiting. Exiting."
exit 1
```

Make executable:

```bash
chmod +x ~/set-default-sink.sh
sudo loginctl enable-linger grakovne
```

---

## Step 3: Create systemd User Unit

Save as `~/.config/systemd/user/set-sink.service`:

```ini
[Unit]
Description=Set default sink to AirPlay
After=pipewire.service

[Service]
ExecStart=/home/grakovne/set-default-sink.sh
Restart=on-failure

[Install]
WantedBy=default.target
```

---

## Step 4: Enable the Unit

```bash
systemctl --user daemon-reexec
systemctl --user daemon-reload
systemctl --user enable --now set-sink.service
```

---

## Optional Debugging

Check default sink:

```bash
pactl info | grep "Default Sink"
```

List available sinks:

```bash
pactl list short sinks
```
