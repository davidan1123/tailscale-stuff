# Reaching a lab web GUI through SSH

How to open a web GUI on the dojo lab network from the laptop by tunnelling
through seat8 over SSH. This works with no Tailscale subnet route: SSH reaches
seat8 by its Tailscale name, and seat8 reaches the lab network directly.

```
browser ──> localhost:8443 ──SSH tunnel──> seat8 ──> 172.16.44.203:443
```

## One web GUI: local port forward

```bash
ssh -N -L 8443:172.16.44.203:443 seat8@seat8
```

Then open **https://localhost:8443** in the browser.

| Part | Meaning |
|---|---|
| `8443` | Any free port on the laptop |
| `172.16.44.203:443` | The GUI's address as seen from seat8 (use `:80` and `http://` if it isn't HTTPS) |
| `seat8@seat8` | User `seat8` at seat8's Tailscale MagicDNS name |
| `-N` | Forward only, no shell. Ctrl+C closes the tunnel |

If the GUI runs on seat8 itself, use `-L 8443:localhost:443`.

The browser will likely warn about the certificate because the name
(`localhost`) doesn't match the device's certificate; that is expected.

## Several GUIs: SOCKS proxy

One tunnel for every device on the lab network:

```bash
ssh -N -D 1080 seat8@seat8
```

In Firefox: Settings → Network Settings → Manual proxy → SOCKS Host
`localhost`, Port `1080`, SOCKS v5, and tick **Proxy DNS when using SOCKS v5**.
Then browse straight to `https://172.16.44.203` or any other lab address.

Tip: use a separate Firefox profile for the lab so normal browsing isn't sent
through the proxy (`firefox -P`).

## Save it in `~/.ssh/config`

```
Host dojo-gui
    HostName seat8
    User seat8
    IdentityFile ~/.ssh/dojo
    IdentitiesOnly yes
    LocalForward 8443 172.16.44.203:443
    # or, for the SOCKS proxy instead:
    # DynamicForward 1080
```

Then open the tunnel with:

```bash
ssh -N dojo-gui
```

## Password prompts

If SSH keeps asking for a password, the server isn't accepting the key yet.
Install it once (asks for the password one last time):

```bash
ssh-copy-id -i ~/.ssh/dojo.pub seat8@seat8
```

Test key-only login:

```bash
ssh -o PreferredAuthentications=publickey seat8@seat8
```

If it is still rejected, check on the server that `~/.ssh/authorized_keys`
holds the key and that `~`, `~/.ssh` and `authorized_keys` aren't writable by
others (`chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys && chmod go-w ~`).

## Troubleshooting

| Problem | Check |
|---|---|
| `bind: Address already in use` | Pick another local port, or find the user: `ss -ltnp \| grep 8443` |
| Page never loads | From seat8, test the GUI: `curl -kI https://172.16.44.203` |
| `channel ... open failed: administratively prohibited` | The server disallows forwarding (`AllowTcpForwarding no` in `sshd_config`) |
