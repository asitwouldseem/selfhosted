# Initial Pi Setup 
This Pi serves several purposes. As a lower-power machine, it's used to send NUT commands to other machines protected by UPS circuits. It's also used as a backup Pi-hole for when Proxmox is offline and as the NTP server for my homelab.

### 1. Update Pi
```
sudo apt update
sudo apt upgrade
sudo rpi-update
```

## GPS Time Server
Install packages.
`sudo apt install pps-tools gpsd gpsd-clients chrony`

### 2. Make sure serial port is enabled

`sudo raspi-config` > Interface Options > Serial Interface

### 3. Update config.txt

```
sudo nano /boot/firmware/config.txt

# GPS Time with GT-U7 Module
dtoverlay=pps-gpio,gpiopin=18
enable_uart=1
init_uart_baud=9600
```

### 4. Update /etc/modules

`'pps-gpio'`

### 5. Reboot, check if pps is working. 

`lsmod | grep pps`

### 6. Edit etc/default/gpsd 

```
START_DAEMON="true"
USBAUTO="true"
DEVICES="/dev/serial0 /dev/pps0"
GPSD_OPTIONS="-n"
```

### 7. Check output with `gpsmon`

### 8. Check for PPS 

`sudo ppstest /dev/pps0`

### 9. Add two lines to /etc/chrony/chrony.conf

`refclock SHM 0 refid NMEA offset 0.200`
`refclock PPS /dev/pps0 refid PPS lock NMEA`

### 10. Restart Chrony

`sudo systemctl restart chrony`

### 11. Confirm PPS is a source in Chrony

`chronyc sources`

NTP runs on port 123. Remember to allow in ufw.
