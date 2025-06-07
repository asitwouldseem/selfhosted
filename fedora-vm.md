# Initial Fedora VM setup 

### 0. Update the repositories.
`sudo dnf update -y `

### 1. Install Cockpit Podman and nano (sorry!)
`sudo dnf install -y cockpit-podman nano`

### 2. Change Cockpit Port (Lyrion will need 9090)

First, we'll need to make the directory for the configuration file.
`sudo mkdir /etc/systemd/system/cockpit.socket.d/`

Then modify the configuration: 
```
sudo nano /etc/systemd/system/cockpit.socket.d/listen.conf

[Socket]
ListenStream=
ListenStream=9002
```

### 3. Update SELinux Policy, otherwise it won't allow us to run Cockpit on a new port.
`sudo semanage port -m -t websm_port_t -p tcp 9002`

```
sudo systemctl daemon-reload
sudo systemctl restart cockpit.socket
```

### 4. Update firewalld
```
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=9002/tcp
sudo firewall-cmd --permanent --zone=FedoraServer --remove-port=9090/tcp
sudo firewall-cmd --reload
```

### 5. Move to passwordless login.
I really like the Yubikey hardware keys for storing my SSH keys. This is where I'd usually opt for authentication that way.

### 6. Setup containers
 - Mosquito
 - MariaDB
 - Valetudo2PNG
 - Lyrion Media Server
 - TVHeadEnd

We're going to create an 'apps' directory in the home folder.

`mkdir ~/apps`

Because Podman runs containers rootless, containers run as the user rather than root. This means we need to change the ownership to the user account (e.g.) `sudo chown cameron -R mqtt` and then run (e.g.) `podman unshare chown 200:200 -R mqtt` to allow Podman to access the dirs.

## Mosquito
Mosquito is an MQTT broker. It's pretty neat. I've used ZHA, DeCONZ and Zigbee2MQTT over the years for controlling Zigbee based devices. ZHA and Zigbee2MQTT are (in my controversial opinion at least) about on par as far as features go. Because I already run an MQTT broker for other services and opting for Zigbee2MQTT gives me the opportunity to position the coordinator more centrally, I use it over ZHA. You can fight that out in the comments.

Because we have multiple containers that need to chat locally. Create a Pod within Cockpit that respects: 
- 0.0.0.0:1883 → 1883/tcp (MQTT)
- 0.0.0.0:3000 → 3000/tcp (ValetudoPNG)

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
	--name=mqtt
	--pod mqtt
	--restart=unless-stopped
	-v "/home/../apps/mqtt/config":"/mosquitto/config":Z
	-v "/home/../apps/mqtt/data":"/mosquitto/data":Z
	-v "/home/../apps/mqtt/log":"/mosquitto/log":Z
docker.io/eclipse-mosquitto:latest
```

Make sure MQTT is added: 
`sudo firewall-cmd --permanent --zone=FedoraServer --add-service=mqtt`
`sudo firewall-cmd --reload`

Usually at this point, I'd confirm the container boots up fine from Cockpit and add it to Home Assistant. If that's good, I'll move onto the next container. 

## MariaDB
MariaDB is a drop-in replacement for MySQL. It's my favourite of the relational databases to work with. I host this largely for Home Assistant and keep the container here in case I choose to use MariaDB for other services I self-host. SQLite is probably enough for 90% of people.

Like other containers, we'll create a directory first. Then chown/unshare to allow Podman to access.

`mkdir ~/apps/sql/`

And then we'll spin up a container. 

```
podman run -d
	--hostname=sql.local
	--name=sql
	--restart=unless-stopped
        --env MARIADB_RANDOM_ROOT_PASSWORD=1
	--env MARIADB_USER=secret_user
	--env MARIADB_PASSWORD=secret_pass
	--env MARIADB_DATABASE=db_haos
	-p 3306:3306
	-v "/home/../apps/sql":"/var/lib/mysql":Z
docker.io/mariadb:lts
```

Make sure SQL is added to the firewall: 
`sudo firewall-cmd --permanent --zone=FedoraServer --add-service=mysql`
`sudo firewall-cmd --reload`

Bing, bang. Done! Add the config into Home Assistant and keep on chugging along. This is formatted like: 

```
# Use locally hosted DB server rather than SQLite
recorder:
  db_url: mysql://user:password@SERVER_IP/DB_NAME?charset=utf8mb4 (use a secret!)
```

# ValetudoPNG
I use [Valetudo](https://valetudo.cloud/) firmware on my robot vacuum. To display the map, we need to run a seperate service to generate the neccesary data for Home Assistant. There's a few ways to do this, but this is my preferred approach. We should be on a roll now...

Create your directory, [touch the yaml](https://github.com/erkexzcx/valetudopng/blob/main/config.example.yml), chown/unshare, etc!

```
podman run 
    -d
    -v "/home/../apps/valetudopng/config.yml":"/config.yml":z
    --pod mqtt
    --name=valetudopng
    --restart unless-stopped
    ghcr.io/erkexzcx/valetudopng:latest
```

```
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=3000/tcp
sudo firewall-cmd --reload
```

Woohoo!

# Lyrion Media Server
Because I use Fedora Server, SELinux runs by default. To allow SMB shares to be mapped inside Podman, we need to first allow it: 
`sudo setsebool virt_use_samba on`
