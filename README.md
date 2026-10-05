# Music Assistant + Sendspin BT Bridge

Two services in one compose stack:
- **Music Assistant** — `http://<host>:8095`
- **Sendspin BT Bridge** (web UI) — `http://<host>:8094`

## Prerequisites for Sendspin

Sendspin requires PulseAudio and a Bluetooth adapter on the host.

### 1. Install PulseAudio with Bluetooth module

```bash
sudo apt-get install -y pulseaudio pulseaudio-module-bluetooth
```

### 2. Enable user session to persist without a login (linger)

```bash
loginctl enable-linger $USER
```

### 3. Start PulseAudio on headless systems

PulseAudio must keep running without clients: if it exits on idle, the Bluetooth sinks go away until the next client
connects. How to set that up depends on whether the distro ships a user unit for it.

**Stock user unit present** (`systemctl --user cat pulseaudio.service` prints it — e.g. Ubuntu 26.04). With linger it
starts on its own; only turn off the idle exit with a drop-in:

```bash
mkdir -p ~/.config/systemd/user/pulseaudio.service.d

cat > ~/.config/systemd/user/pulseaudio.service.d/override.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/pulseaudio --daemonize=no --exit-idle-time=-1 --log-target=journal
EOF

systemctl --user daemon-reload
systemctl --user enable pulseaudio.socket pulseaudio.service
systemctl --user restart pulseaudio.service
```

Don't add the service below next to the stock one: both would fight for the same socket.

**No stock unit** — on servers without a desktop environment PulseAudio may not start via systemd at all.
Create a user service that bypasses the missing audio hardware dependency:

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/pulseaudio-headless.service << 'EOF'
[Unit]
Description=PulseAudio (headless)
After=dbus.socket

[Service]
ExecStart=/usr/bin/pulseaudio --daemonize=no --exit-idle-time=-1 --log-target=journal
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable pulseaudio-headless.service
systemctl --user start pulseaudio-headless.service
```

Verify:
```bash
pactl info | head -1
# Expected: Server String: /run/user/<UID>/pulse/native
```

### 4. Set AUDIO_UID in .env

```bash
id -u
# Put the result in .env: AUDIO_UID=<value>
```

## Start

```bash
docker compose up -d
```

**No Bluetooth adapter?** Start only Music Assistant: `docker compose up -d music-assistant`. Without an adapter
`bluetooth.service` is skipped at boot (`ConditionPathIsDirectory=/sys/class/bluetooth`) and the bridge's entrypoint
hangs on `bluetoothctl show` for good — the container stays unhealthy. The first start right after
`apt install bluez` still works (apt started bluetoothd), so this only shows up after a reboot.

## Configure

1. Open Music Assistant at `http://<host>:8095` and complete initial setup
2. Open Sendspin at `http://<host>:8094`, set a password and connect to Music Assistant
3. Add your Bluetooth speakers via **Devices → Scan nearby → Pair and Add**
