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

On servers without a desktop environment, PulseAudio will not start via systemd automatically.
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
systemctl --user enable --now pulseaudio-headless.service
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

## Configure

1. Open Music Assistant at `http://<host>:8095` and complete initial setup
2. Open Sendspin at `http://<host>:8094`, set a password and connect to Music Assistant
3. Add your Bluetooth speakers via **Devices → Scan nearby → Pair and Add**
