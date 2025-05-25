# Initial Fedora VM setup 

0. Update the repositories.
sudo dnf update -y 

1. Install Cockpit Podman and nano (sorry!)
sudo dnf install -y cockpit-podman nano

2. Change Cockpit Port (Lyrion will need 9090)

First, we'll need to make the directory for the configuration file.
`sudo mkdir /etc/systemd/system/cockpit.socket.d/`

Then modify the configuration: 
```
sudo nano /etc/systemd/system/cockpit.socket.d/listen.conf

[Socket]
ListenStream=
ListenStream=9002
```

3. Update SELinux Policy, otherwise it won't allow us to run Cockpit on a new port.
`sudo semanage port -m -t websm_port_t -p tcp 9002`

```
sudo systemctl daemon-reload
sudo systemctl restart cockpit.socket
```

4. Update firewalld
```
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=9002/tcp
sudo firewall-cmd --permanent --zone=FedoraServer --remove-port=9090/tcp
sudo firewall-cmd --reload
```

5. Move to passwordless login.
I really like the Yubikey hardware keys for storing my SSH keys. This is where I'd usually opt for authentication that way.

6. Setup containers
 - Mosquito
 - Lyrion Media Server
 - Piper
 - Whisper
 - Valetudo2PNG

We're going to create an 'apps' directory in the home folder.

`mkdir ~/apps`

Because Podman runs containers rootless, containers run as the user rather than root. This means we need to change the ownership to the user account (e.g.) `sudo chown cameron -R mqtt` and then run (e.g.) `podman unshare chown 200:200 -R mqtt` to allow Podman to access the dirs.

### Mosquito
Mosquito is an MQTT broker. It's pretty neat. I've used ZHA, DeCONZ and Zigbee2MQTT over the years for controlling Zigbee based devices. ZHA and Zigbee2MQTT are (in my controversial opinion at least) about on par as far as features go. Because I already run an MQTT broker for other services and opting for Zigbee2MQTT gives me the opportunity to position the coordinator more centrally, I use it over ZHA.

`mkdir ~/apps/mqtt/config ~/apps/mqtt/data ~/apps/mqtt/log`

We'll also need a config file to get us started. 

`nano ~/apps/mqtt/config/mosquitto.conf`

```
# General settings
persistence true
persistence_location /mosquitto/data/

# Logging
log_dest file /mosquitto/log/mosquitto.log
log_type error
log_type warning
listener 1883

# Setup authentication
allow_anonymous false
password_file /mosquitto/config/password.txt
```

Before we go any further, I want to generate a password for Home Assistant to chat securely with the MQTT broker. 

`sudo dnf install mosquitto_passwd`

`sudo mosquitto_passwd -c ~/apps/mqtt/config/password.txt username`

Give the user a password. Then, change ownership / unshare. And run the following command:

```
podman run -d
	--hostname=mqtt.local
	--name=mqtt
	--restart=unless-stopped
	-p 1883:1883
	-v "/home/../apps/mqtt/config":"/mosquitto/config":Z
	-v "/home/../apps/mqtt/data":"/mosquitto/data":Z
	-v "/home/../apps/mqtt/log":"/mosquitto/log":Z
docker.io/eclipse-mosquitto:latest
```

Make sure MQTT is added: 
`sudo firewall-cmd --permanent --zone=FedoraServer --add-service=mqtt`
`sudo firewall-cmd --reload`

Usually at this point, I'd confirm the container boots up fine from Cockpit and add it to Home Assistant. If that's good, I'll move onto the next container. 
