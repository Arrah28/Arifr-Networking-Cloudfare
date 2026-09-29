# Cloud Networking: NGINX on EC2 behind a Cloudflare domain

A public web server on AWS, reachable over the internet through a custom domain. Small project, but it covers the whole path a request takes: DNS → Cloudflare → security group → EC2 → NGINX.

```
Browser ──DNS──▶ Cloudflare (A record, proxied) ──▶ EC2 public IPv4 ──▶ Security group (80/443) ──▶ NGINX on Amazon Linux 2023
```

## What was built

| Layer | Detail |
|---|---|
| **Compute** | Amazon EC2 running Amazon Linux 2023 |
| **Web server** | NGINX, installed with `dnf`, enabled as a systemd service |
| **Firewall** | AWS security group allowing inbound TCP 80 and 443 from `0.0.0.0/0` |
| **DNS** | Cloudflare-managed domain with an A record pointing at the instance's public IPv4 |

| EC2 configuration | Security group | Cloudflare DNS | Result |
|---|---|---|---|
| ![](Networking/Screenshots/EC2%20Instance%20Config.png) | ![](Networking/Screenshots/AWS%20SG%20config.png) | ![](Networking/Screenshots/CloudFlare%20Config.png) | ![](Networking/Screenshots/Welcome%20NGNIX.png) |

## Steps

```bash
# on the instance (Amazon Linux 2023 uses dnf, not yum)
sudo dnf install -y nginx
sudo systemctl enable --now nginx
curl localhost          # confirm it answers locally before touching DNS
```

1. Launch the instance and attach a security group that allows 80 and 443 inbound.
2. Install and start NGINX; verify with `systemctl status nginx` and `curl localhost`.
3. Test from outside using the **public IP over plain `http://`** before adding DNS.
4. In Cloudflare, add an A record for the domain pointing at the public IPv4.
5. Confirm the NGINX welcome page loads on the domain.

## Troubleshooting: the Cloudflare 521

The domain initially returned a **Cloudflare 521 (web server is down)**. Working it out in order:

| Suspect | Check | Outcome |
|---|---|---|
| DNS not propagated | A record resolved to the right IP | Fine |
| Security group blocking 80 | Inbound rules on the SG | Fine |
| NGINX not actually running | `systemctl status nginx` via EC2 Instance Connect | **This was it**: the user-data script hadn't started the service |
| Browser forcing HTTPS / HSTS cache | Tested with explicit `http://` and the raw public IP | Confirmed it wasn't the browser |

A 521 means Cloudflare reached the origin's IP but nothing answered on the port. It's an origin problem, not a DNS problem, which narrows it down fast once you know that.

## What I learned

- **Verify the service, don't assume user-data worked.** `systemctl status` immediately after launch would have saved most of the debugging time.
- **Test by public IP before adding DNS or a proxy**, so each layer is proven before the next one is added.
- **Set Cloudflare SSL/TLS mode deliberately** when the orange-cloud proxy is on, otherwise HTTPS can fail even though HTTP works.
- Security groups are stateful and deny by default; port 80 has to be opened explicitly.

## Repository layout

```
Networking/Project-Overview.md   Objectives, architecture and implementation steps
Networking/Notes.md              Learning log: mistakes, fixes and best practices
Networking/Screenshots/          Console and browser evidence
```
