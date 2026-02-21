
# Testing gluetun container setup

```shell
$ docker compose --file=docker-compose.yml up --detach
$ docker network create --scope=local --attachable --subnet=192.168.12.0/24 traefik_public
$ docker network create --scope=local --attachable --subnet=192.168.13.0/24 vpn
$ docker-compose logs openvpn
$ docker-compose logs openvpn
...
...
$ 
$ docker-compose ps
NAME              IMAGE                                          COMMAND                  SERVICE           CREATED          STATUS          PORTS
dozzle            amir20/dozzle:latest                           "/dozzle"                dozzle            13 minutes ago   Up 13 minutes   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
openvpn           media.johnson.int:5000/docker-gluetun:latest   "/init"                  openvpn           11 minutes ago   Up 11 minutes   0.0.0.0:8168->8168/tcp, [::]:8168->8168/tcp
portainer         portainer/portainer-ce:sts                     "/portainer -H tcp:/…"   portainer         13 minutes ago   Up 13 minutes   8000/tcp, 9443/tcp, 0.0.0.0:9010->9000/tcp, [::]:9010->9000/tcp
portainer-agent   portainer/agent:latest                         "./agent"                portainer-agent   13 minutes ago   Up 13 minutes   
downloader        ghcr.io/linuxserver/downloader:latest         "/init"                  downloader       11 minutes ago   Up 11 minutes   
socket-proxy      tecnativa/docker-socket-proxy:latest           "docker-entrypoint.s…"   socket-proxy      13 minutes ago   Up 13 minutes   0.0.0.0:2375->2375/tcp, [::]:2375->2375/tcp
traefik           traefik:v3.6.1                                 "/entrypoint.sh trae…"   traefik           13 minutes ago   Up 13 minutes   0.0.0.0:80->80/tcp, [::]:80->80/tcp, 0.0.0.0:443->443/tcp, [::]:443->443/tcp
whoami            containous/whoami                              "/whoami"                whoami            13 minutes ago   Up 13 minutes   0.0.0.0:9080->80/tcp, [::]:9080->80/tcp
$ 
$ docker network inspect traefik_public | jq '.[0].IPAM.Config[0].Subnet'
"192.168.12.0/24"
$ 
```

## Confirm Using the VPN (Important Checks)

### Check the IP the torrent gets

How to Verify (Step-by-Step Tests)
Run these while a torrent is active (or add a popular legal test torrent).

#### Test 1: Check what IP the torrent swarm sees (most important)

Go to https://torguard.net/checkmytorrentipaddress.php
Copy the generated magnet link
Add it to qBittorrent
Wait 30–120 seconds
Look at the "Error" or "Status" column — it should show only your NordVPN IP (or "No direct connections" if no incoming).

If it shows 172.74.37.27 (or anything not NordVPN) → leak confirmed.
Alternative sites:

https://ipleak.net (use "Torrent Address Detection" magnet)
https://www.checkmytorrentip.net (magnet link test)

#### Test 2: Confirm qBittorrent's outgoing IP

Inside qBittorrent container:

```shell
$ docker exec -it qbittorrent-container-name sh
/$ curl -s https://api.ipify.org
# or
/$ curl -s ifconfig.me
```
Should return NordVPN IP. If it returns your ISP IP → outgoing traffic not routed via VPN.

Run:
```shell
$ docker exec downloader curl -s https://ifconfig.me
192.145.117.83
$ 
$ docker exec downloader curl -s https://ipinfo.io/json
{
  "ip": "192.145.117.83",
  "city": "Charlotte",
  "region": "North Carolina",
  "country": "US",
  "loc": "35.2271,-80.8431",
  "org": "AS141039 PacketHub S.A.",
  "postal": "28202",
  "timezone": "America/New_York",
  "readme": "https://ipinfo.io/missingauth"
}
$ docker exec openvpn curl -s https://ifconfig.me
192.145.117.83
$ 
```

Compare the ifconfig.me IP:

- 192.145.117.83 ← this is not your home/public IP
- In the OpenVPN logs, the connection was made to:
```text
Selected: United States #8497 (us8497.gluetun.com) at 192.145.117.146
```

→ The remote VPN server IP is 192.145.117.146
→ Your assigned outbound IP 192.145.117.83 is clearly in the same /24 subnet as the VPN exit node.

This is evidence that traffic is exiting via gluetun.
Most residential ISPs give you something in 100-120.x.x.x, 70-80.x.x.x, 24/50/73/ etc. ranges — not 192.145.x.x.

## Confirm downloader has no direct attachment to traefik_public

Run this:
```shell
$ docker inspect downloader --format '{{json .NetworkSettings.Networks }}'
{}
$
```

You should see empty object {} or just the special "service container" reference — not an entry for traefik_public (192.168.12.0/24).

Expected output looks roughly like:
```json
{}
```

or sometimes shows only loopback / internal info — but no 192.168.12.x address.

Compare with a normal container (e.g. traefik itself):

```
docker inspect traefik --format '{{json .NetworkSettings.Networks }}'
```

→ You'll see it has an IP in 192.168.12.0/24.

#### Test 3: Check listening interfaces inside qBittorrent container

```shell
$ docker exec -it qbittorrent-container-name sh
/$ netstat -tuln | grep 6881
/$ ss -tuln | grep 6881
```

- Bad (leak): 0.0.0.0:6881 or :::6881
- Good: 10.x.x.x:6881 (VPN tunnel IP range) or nothing (if you disable incoming)

#### Test 4: Simulate VPN drop (kill-switch test)

- Stop the OpenVPN container (docker stop openvpn)
- Try to start a torrent in qBittorrent or run curl https://api.ipify.org inside qBittorrent container 
- Should fail completely (no connection) if kill-switch works 
- If it succeeds → no kill-switch or rules ineffective


## Strongest / cleanest confirmation commands

Run these from the host:

1. Check public IP **from inside downloader**
```shell
docker exec downloader curl -s https://ifconfig.me
# or better — use a service that also shows geo
docker exec downloader curl -s https://ipinfo.io/json
```

Look for "org": "gluetun" / "city", "region", etc. — should match US server.

2. Check public IP from inside openvpn container (should be same as downloader)
```shell
docker exec openvpn curl -s https://ifconfig.me
```

→ Expect exactly the same IP as downloader.

3. Check public IP from a normal container (e.g. whoami or traefik)
```
docker exec whoami wget -qO- https://ifconfig.me
# or
docker exec traefik curl -s https://ifconfig.me
```

→ This should show your real public/home IP (not the gluetun one).

If 1 and 2 match each other → and are different from 3 → routing is working perfectly.


| Container | Network mode | Has own IP on traefik_public? | Outbound traffic goes via |
| :--- | :--- | :--- | :--- |
| openvpn | default bridge + vpn net | yes | gluetun tunnel |
| downloader | service:openvpn | no | same net namespace as openvpn → via gluetun |
| traefik, whoami, dozzle, etc. | default / traefik_public | yes | host → real internet |

So downloader does not have an interface on traefik_public (192.168.12.0/24) at all — it shares the network stack of the openvpn container.

## Bonus: What you should not do

Do not add downloader to the traefik_public network — it would break the VPN routing (split-tunnel / leak possible).

The current network_mode: service:openvpn + depends_on: openvpn + ports published on openvpn is the standard & correct pattern.

If you want even more certainty, just run the three curl commands above and compare the IPs — that removes any doubt.
