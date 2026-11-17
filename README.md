# Tailscale subnet routing: reaching the dojo lab network

Notes from troubleshooting why the laptop (`engineer`) could SSH to the dojo
machine (`seat8`) over Tailscale but could not reach other services on the
same lab network.

Tailscale version: 1.104.7 (Linux client).

See also: [Reaching a lab web GUI through SSH](ssh-web-gui-access.md).

## Symptom

- `ssh` to seat8 over Tailscale works.
- Other hosts/services on the lab network (e.g. `172.16.44.203`) are unreachable.
- The route for the lab network goes out through the Wi-Fi router, not Tailscale:

```
$ ip route get 172.16.44.203
172.16.44.203 via 192.168.10.1 dev wlp131s0f0 src 192.168.10.229 uid 1000
```

## Cause

Two separate things have to be true for subnet routing to work, and neither was:

1. **The node on the lab network must advertise the subnet.** seat8 was on the
   tailnet only as itself (`100.122.11.124`) and as an exit node; it was not
   sharing the lab network, so no other tailnet device could route to it.
2. **The client must accept advertised routes.** On Linux, `--accept-routes`
   is off by default. Tailscale's health check said so directly:

   ```
   # Health check:
   #     - Some peers are advertising routes but --accept-routes is false
   ```

## Fix

### 1. On the subnet router (seat8)

Confirm the lab subnet first:

```bash
ip -4 addr
```

Enable IP forwarding (persistent):

```bash
echo -e 'net.ipv4.ip_forward = 1\nnet.ipv6.conf.all.forwarding = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf && sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Advertise the subnet, keeping the existing exit-node offer:

```bash
sudo tailscale set --advertise-routes=172.16.44.0/24 --advertise-exit-node
```

### 2. In the Tailscale admin console

Machines → seat8 → **Edit route settings** → approve the subnet
(`172.16.44.0/24`).

Check from any client that it is approved (`PrimaryRoutes` should list it):

```bash
tailscale status --json | python3 -c 'import json,sys;d=json.load(sys.stdin);[print(p["HostName"],p.get("PrimaryRoutes")) for p in d["Peer"].values()]'
```

### 3. On the client (laptop)

```bash
sudo tailscale set --accept-routes
```

Verify the lab network now goes through Tailscale:

```bash
ip route get 172.16.44.203
```

Expected: `... dev tailscale0 ...`

## Caveat: overlapping home LAN route

The OPNsense node also advertises `192.168.10.0/24`, which is the laptop's own
home LAN. With `--accept-routes` on, Tailscale's routing table (table 52) takes
priority over the main table, so local LAN traffic may get sent over Tailscale
instead of directly over Wi-Fi.

After enabling accept-routes, check:

```bash
ip route get 192.168.10.1
```

- `dev wlp131s0f0` → fine.
- `dev tailscale0` and local devices/internet misbehave → either turn it off
  again (`sudo tailscale set --accept-routes=false`), stop OPNsense from
  advertising `192.168.10.0/24` while at home, or only enable accept-routes
  when away from home.

## Alternative: skip the subnet route for SSH

To reach seat8 itself, the subnet route is not needed. Use its Tailscale name
in `~/.ssh/config`:

```
Host dojo
    HostName seat8          # Tailscale MagicDNS name instead of the lab IP
    User seat8
    IdentityFile ~/.ssh/dojo
    IdentitiesOnly yes
```

## Quick reference

| Check | Command |
|---|---|
| Peers and health warnings | `tailscale status` |
| Is accept-routes on? | `tailscale debug prefs \| grep RouteAll` |
| Which peers advertise subnets | `tailscale status --json` (see `PrimaryRoutes`) |
| Routes Tailscale installed | `ip route show table 52` |
| Which path a host takes | `ip route get <ip>` |
