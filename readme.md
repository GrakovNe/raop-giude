Plexamp Headless + AirPlay (RAOP) on Raspberry Pi
=================================================

Minimal setup guide for running Plexamp in headless mode and routing all system audio to an AirPlay (RAOP v1) speaker.

* * *

📦 Required Packages
--------------------

Install PulseAudio with RAOP support and device discovery:

`sudo apt update sudo apt install \   pulseaudio \   pulseaudio-utils \   pulseaudio-module-raop \   avahi-daemon`

* * *

🛠 Optional: Disable PipeWire (if it interferes)
------------------------------------------------

`systemctl --user mask pipewire.service pipewire.socket wireplumber.service`

* * *

🔁 Enable user lingering (so services persist after reboot)
-----------------------------------------------------------

`sudo loginctl enable-linger grakovne`

* * *

🚀 Plexamp Headless systemd user unit
-------------------------------------

File: `~/.config/systemd/user/plexamp.service`

    [Unit]
    Description=Plexamp Headless
    After=network-online.target
    
    [Service]
    Type=simple
    User=grakovne
    Environment=NODE_ENV=production
    Environment=XDG_RUNTIME_DIR=/run/user/1000
    WorkingDirectory=/opt/plexamp
    ExecStart=/usr/local/n/versions/node/20.19.0/bin/node js/index.js
    Restart=always
    
    [Install]
    WantedBy=default.target
    

* * *

🔈 RAOP auto-setup script
-------------------------

File: `~/set-default-raop.sh`

    #!/bin/bash
    
    if ! pactl list short modules | grep -q module-raop-discover; then
        pactl load-module module-raop-discover
        echo "Loaded module-raop-discover"
    fi
    
    for i in {1..30}; do
        SINK=$(pactl list short sinks | grep raop | awk '{print $2}' | head -n1)
        if [[ -n "$SINK" ]]; then
            pactl set-default-sink "$SINK"
            echo "Default sink set to: $SINK"
    
            for j in {1..5}; do
                INPUT_ID=$(pactl list short sink-inputs | grep "$SINK" | awk '{print $1}' | head -n1)
                if [[ -n "$INPUT_ID" ]]; then
                    pactl set-sink-input-volume "$INPUT_ID" 50%
                    echo "Sink-input $INPUT_ID volume set to 50%"
                    break
                fi
                sleep 1
            done
    
            exit 0
        fi
        sleep 1
    done
    
    echo "RAOP sink not found"
    exit 1
    

* * *

🧩 systemd user unit for RAOP auto-setup
----------------------------------------

File: `~/.config/systemd/user/set-raop-default.service`

    [Unit]
    Description=Set default RAOP sink (AirPlay) if available
    After=sound.target network-online.target
    Requires=plexamp.service
    
    [Service]
    Type=oneshot
    ExecStart=/home/grakovne/set-default-raop.sh
    
    [Install]
    WantedBy=default.target
    

* * *

✅ Final steps
-------------

`systemctl --user daemon-reload systemctl --user enable plexamp.service systemctl --user enable set-raop-default.service systemctl --user start plexamp.service`
