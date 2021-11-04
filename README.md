# Raspberry Pi as Bluetooth-MIDI Repeater

## Topology



## Tested components

- **Raspberry Pi 3** Model B v1.2
- 2x **WIDI BUD** (BLE to USB MIDI Bridge)
- 2x **Wave for Music** (Genki Instruments), firmware version 1.7.0, **standalone mode**

## Tested system

Raspberry Pi OS Lite \
Release date: May 7th 2021 \
Kernel version: 5.10

## Basic installation

- Download original [RPi Imager](https://www.raspberrypi.com/software/)
- Write "Raspberry Pi OS Lite" to SD card
- Boot image
- Login (user: `pi`, password: `raspberry`)
- Basic Setup - `sudo raspi-config`
  - Wifi
    - `1 System Options`
    - `S1 Wireless LAN`
  - Password
    - `1 System Options`
    - `S3 Password`
  - SSH
    - `3 Interface Options`
    - `P2 SSH`
- Check IP on Wifi adapter: `ip a`
- Connect through SSH (optional) - better for copy-pasting commands
- Update
  - `sudo apt update`
  - `sudo apt upgrade -y`
- Setup DHCP
  - `sudo echo "interface eth0" >> /etc/dhcpcd.conf`
  - `sudo echo "static ip_address=192.168.120.1/24" >> /etc/dhcpcd.conf`
  - `sudo reboot`
  - Login again
  - `sudo apt install dnsmasq -y`
  - `sudo nano /etc/dnsmasq.conf` add lines
    ```
    interface=eth0
    bind-dynamic
    dhcp-range=192.168.120.100,192.168.120.200,255.255.255.0,12h
    dhcp-option=3
    dhcp-option=6
    ```
  - `sudo service dnsmasq restart`

## Test Wave + WiDi

- Insert WiDi dongels
- Start Waves - they should connect in one minutes
- Run `aseqdump -l`. Expected output:
  ```
   Port    Client name                      Port name
    0:0    System                           Timer
    0:1    System                           Announce
   14:0    Midi Through                     Midi Through Port-0
   24:0    WidiBud                          WidiBud MIDI 1
   28:0    WidiBud                          WidiBud MIDI 1
  ```
- (Alternative) Run `amidi -l`. Expected output:
  ```
  Dir Device    Name
  IO  hw:2,0,0  WidiBud MIDI 1
  IO  hw:3,0,0  WidiBud MIDI 1
  ```
- (Alternative) Run `aplaymidi -l`. Expected output:
  ```
   Port    Client name                      Port name
   14:0    Midi Through                     Midi Through Port-0
   24:0    WidiBud                          WidiBud MIDI 1
   28:0    WidiBud                          WidiBud MIDI 1
  ```
- Warning: This outputs represents installed dongles, not connected Waves \
  Blicking 2 dots in the middle row indicate disconnected Wave
- Test incoming events. Run `aseqdump --port 24:0,28:0` or any other port from
  previus outputs. It shows you incoming MIDI CC messages from Waves.
  For example:
  ```
  ...
    28:0   Control change          0, controller 50, value 63
    28:0   Control change          0, controller 52, value 101
    24:0   Control change          0, controller 17, value 89
    24:0   Pitch bend              0, value 20
    28:0   Control change          0, controller 52, value 100
    24:0   Control change          0, controller 16, value 84
    24:0   Pitch bend              0, value 0
  ...
  ```

## Install core

Follow compiling instructions: [https://blog.tarn-vedra.de/pimidi-box/](https://blog.tarn-vedra.de/pimidi-box/)

## Config files

`/etc/raveloxmidi_hw2.conf`

```
alsa.input_device = hw:2,0,0 
alsa.output_device = hw:2,0,0 
network.bind_address = 0.0.0.0 
service.name = Wave-RPi-hw2 
network.control.port = 5004
network.data.port = 5005
network.local.port = 5006
```


`/etc/raveloxmidi_hw3.conf`

```
alsa.input_device = hw:3,0,0 
alsa.output_device = hw:3,0,0 
network.bind_address = 0.0.0.0 
service.name = Wave-RPi-hw3
network.control.port = 5007
network.data.port = 5008
network.local.port = 5009
```

## Service files

`/etc/systemd/system/raveloxmidi_hw2.service`
```
[Unit]
After=local-fs.target network.target
Description=raveloxmidi RTP-MIDI network server HW2

[Install]
WantedBy=multi-user.target

[Service]
ExecStart=/usr/local/bin/raveloxmidi -N -c /etc/raveloxmidi_hw2.conf

Type=simple
Restart=always
```


`/etc/systemd/system/raveloxmidi_hw3.service`
```
[Unit]
After=local-fs.target network.target
Description=raveloxmidi RTP-MIDI network server HW3

[Install]
WantedBy=multi-user.target

[Service]
ExecStart=/usr/local/bin/raveloxmidi -N -c /etc/raveloxmidi_hw3.conf

Type=simple
Restart=always
```

Enable services:
```
sudo systemctl daemon-reload
sudo systemctl enable raveloxmidi_hw2.service
sudo systemctl start raveloxmidi_hw2.service
sudo systemctl enable raveloxmidi_hw3.service
sudo systemctl start raveloxmidi_hw3.service
```