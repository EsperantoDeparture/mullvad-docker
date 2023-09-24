# Mullvad Docker

This is a tutorial on how to create a docker container that you can use to route your host's traffic using WireGuard with Mullvad VPN.

1. Create a new docker network (you may use any name, just don't forget to use the new name in the following instructions).

```bash
docker network create --subnet 172.20.0.0/24 wgnet
```

2. Create a docker-compose file with a wireguard container, with the appropiate permissions to make network changes and is using the network we created in step 1. Place this file in a directory of your choosing, I personally created one in my home directory.

```yaml
version: '3.5'

services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - NET_RAW
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./conf:/config
    networks:
      default:
        ipv4_address: 172.20.0.50
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
    restart: unless-stopped
    devices:
      - /dev/net/tun:/dev/net/tun
networks:
  default:
    name: wgnet
    external: true

```

Note that we gave the container a static IP manually. This is because docker-compose start up containers, it doesn't check if the IP its assigning was reserved by another container already in the network. In other words, if we have 10 containers listed in the same compose yaml and some have static IPs defined and some don't, docker compose will start assigning IPs to dynamic ones starting with "2". You might want to change your wireguard containers' ip if you plan to run more than 48 containers alongside it.

3. Log into Mullvad and navigate to [Wireguard configuration](https://mullvad.net/en/account/wireguard-config) and follow the instructions to generate the configuration file. After you download this file, unzip it, rename the config file "wg0.conf". Then, inside the directory where you put your compose file create a "conf" directory and place your configuration file there.

Start your container:

```bash
docker-compose up -d

```

Check that the container is running and set up the VPN correctly:

```bash
docker logs wireguard

```

4. Currently, only the container is routing traffic through WireGuard, we need to add some ip route changes:

Before you run the following command, replace VPN_ENDPOINT with your VPN endpoint in wg0.conf.
```bash
sudo ip route del default
sudo ip route add VPN_ENDPOINT via 192.168.1.1
sudo ip route add default via 172.20.0.50
```


The first command removes the default route. The second command allows your container to connect to Mullvad's server in order to establish the tunnel. Finally, the third command routes all traffic via the container.

5. The routing table is generated dynamically on reboot, so all the configuration above will be lost unless we recreate them after a restart.

5.1 Let's create a new service file, /lib/systemd/system/iproute.service with these contents:

```conf
[Unit]
Description=Route everything through WireGuard
After=docker.service

[Service]
Type=oneshot
Restart=on-failure
ExecStart=ip route del default
ExecStart=ip route add VPN_ENDPOINT via 192.168.1.1
ExecStart=ip route add default via 172.20.0.50

[Install]
WantedBy=multi-user.target
```
Be sure to replace VPN_ENDPOINT with your VPN Endpoint in wg0.conf.

Restart systemd's daemon and enable the new service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable iproute.service
```

5.2 In case you lose connection after your PC wakes up from sleep, check your routes:

```bash
ip route show
```

If you see two default routes, it means your network manager is creating a default route every time it enables your network interface. This happened to me, so I had to create another service for this, /etc/systemd/system/remove-default-route.service.

```conf
[Unit]
Description=Remove stupid ip route
After=network-online.target docker.service suspend.target
Wants=network-online.target

[Service]
Type=oneshot
Restart=on-failure
ExecStartPre=/bin/bash -c "while ! /sbin/ip route | /bin/grep -q 'default via 192.168.1.1'; do /bin/sleep 1; done"
ExecStart=/home/esperanto/Scripts/route-fix-after-sleep.sh

[Install]
WantedBy=suspend.target
```

Create a script with these contents:

```bash
#!/bin/bash

# Now, modify the routes
ip route del default via 192.168.1.1
ip route add 154.47.16.34 via 192.168.1.1
cd /home/esperanto/Mullvad && docker-compose stop && docker-compose up -d
```

Replace the ExecStart line to the reference your script and also modify the last line, "cd /home/esperanto/Mullvad", to point to the directory where you saved your docker-compose.yml file.

Finally, restart systemd's daemon and enable the new service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable remove-default-route.service
```