# Pi -> home server via reverse SSH tunnel

```
[Pi @ mom's] --ssh--> [Relay (public IP)] <--reverse tunnel-- [Home server]
```

Both the Pi and the home server sit behind NAT, so neither can reach the other
directly. A small relay with a public IP (cheap VPS, or any machine with a port
forwarded) bridges them. The home server keeps an outbound tunnel open to the
relay; the Pi hops through the relay to reach it.

Placeholders: `RELAY_HOST`, `YOUR_SERVER_USER`.

## 1. Relay
```bash
sudo adduser --disabled-password --gecos "" tunnel
sudo -u tunnel mkdir -p /home/tunnel/.ssh && sudo -u tunnel chmod 700 /home/tunnel/.ssh
```
Add each public key to `/home/tunnel/.ssh/authorized_keys`, locked to the one port:
```
# home server key: may only open the reverse port
restrict,port-forwarding,permitlisten="127.0.0.1:2222" ssh-ed25519 AAAA... home-server
# Pi key: may only forward to that port
restrict,port-forwarding,permitopen="127.0.0.1:2222" ssh-ed25519 AAAA... pi
```
Harden `/etc/ssh/sshd_config`: `PasswordAuthentication no`, `PermitRootLogin no`,
then `sudo systemctl reload ssh`.

## 2. Home server
```bash
sudo apt install autossh
sudo adduser --disabled-password --gecos "" tunnel-client
sudo -u tunnel-client ssh-keygen -t ed25519 -f /home/tunnel-client/.ssh/id_ed25519 -N ""
# put the .pub in the relay's authorized_keys (above), then trust the relay host key:
sudo -u tunnel-client ssh-keyscan RELAY_HOST >> /home/tunnel-client/.ssh/known_hosts
sudo cp home-server-tunnel.service /etc/systemd/system/   # edit RELAY_HOST first
sudo systemctl enable --now home-server-tunnel
```
Also add the Pi's public key to your normal user's `~/.ssh/authorized_keys`
on the home server.

## 3. Pi
```bash
ssh-keygen -t ed25519      # put the .pub in the relay's authorized_keys (above)
cat pi-ssh-config.example >> ~/.ssh/config   # edit placeholders
ssh homeserver
```

## Troubleshooting
- On the relay, `ss -tlnp | grep 2222` should show the listener when the tunnel is up.
- `systemctl status home-server-tunnel` / `journalctl -u home-server-tunnel` on the server.
- `ExitOnForwardFailure` restarts the tunnel if a stale listener holds the port.
