# MBR Networking — Complete Self-Hosted LiveKit Setup Guide

LiveKit API key:
API31ed520e8df9d086

LiveKit API secret:
fdbd0520c6107651b6e910d4ef4e9ac7dd3197b0367aadc74f8f64408bb940eb

read -p "LiveKit API key: " LK_API_KEY
read -s -p "LiveKit API secret: " LK_API_SECRET
echo

TOKEN_ONE=$(lk token create --api-key "$LK_API_KEY" --api-secret "$LK_API_SECRET" --join --room mbr-test-room --identity browser-one --valid-for 1h --token-only)

This guide builds a single-node, production-oriented LiveKit server on an Ubuntu 22.04 LTS virtual machine running under Hyper-V on a Windows 11 Pro host.

It is written for a beginner. Follow the sections in order and do not connect the MBR portal until the standalone LiveKit server passes all verification steps.

## 1. Target architecture

The finished network will look like this:

```text
Internet
   |
   | DNS: YOUR_LIVEKIT_DOMAIN -> 15.235.87.111
   |
Windows 11 Pro host
Public IP: 15.235.87.111
   |
   | Hyper-V internal switch + Windows NAT
   |
Ubuntu 22.04 LTS VM
Static IP: 172.30.0.10
   |
   +-- Caddy: HTTPS/WSS on TCP 443
   +-- LiveKit API: localhost TCP 7880
   +-- LiveKit RTC/TCP: TCP 7881
   +-- LiveKit RTC/UDP mux: UDP 7882
   +-- LiveKit TURN/UDP: UDP 3478
```

The guide uses a single UDP media port (`7882`) instead of LiveKit's default UDP range `50000-60000`. This is easier to forward through the Windows host. LiveKit documents `rtc.udp_port` as its optional UDP-mux port.

## 2. Values used in this guide

| Setting | Value |
|---|---|
| Windows public IP | `15.235.87.111` |
| Hyper-V subnet | `172.30.0.0/24` |
| Windows NAT gateway | `172.30.0.1` |
| Ubuntu VM address | `172.30.0.10` |
| LiveKit internal HTTP port | `7880/TCP` |
| WebRTC TCP fallback | `7881/TCP` |
| WebRTC UDP mux | `7882/UDP` |
| TURN/STUN UDP | `3478/UDP` |

Choose a domain before proceeding. A recommended example is:

```text
meet.mbrnetworking.org
```

Throughout this guide, replace:

```text
YOUR_LIVEKIT_DOMAIN
```

with the real domain, such as `meet.mbrnetworking.org`.

Never put a protocol in DNS. The DNS name is `meet.mbrnetworking.org`, not `https://meet.mbrnetworking.org`.

## 3. Before creating the VM

### 3.1 Confirm required public ports are available

On the Windows host, open **PowerShell as Administrator** and run:

```powershell
Get-NetTCPConnection -State Listen |
  Where-Object LocalPort -In 80,443,7881 |
  Select-Object LocalAddress,LocalPort,OwningProcess
```

Check UDP:

```powershell
Get-NetUDPEndpoint |
  Where-Object LocalPort -In 3478,7882 |
  Select-Object LocalAddress,LocalPort,OwningProcess
```

No output means the ports are currently unused.

If a TCP port is occupied, identify the process:

```powershell
Get-Process -Id THE_OWNING_PROCESS_ID
```

Do not stop an unknown Windows service without first determining what it does.

### 3.2 Create the DNS record

At the DNS provider for `mbrnetworking.org`, create:

```text
Type: A
Name/Host: meet
Value/Target: 15.235.87.111
TTL: 300 or Automatic
Proxy: DNS only, initially
```

If using Cloudflare, select **DNS only** (gray cloud) during initial setup. A proxied record will not carry LiveKit's raw RTC ports.

Confirm DNS from Windows:

```powershell
Resolve-DnsName YOUR_LIVEKIT_DOMAIN
```

It must eventually return:

```text
15.235.87.111
```

DNS may take a few minutes or, depending on the provider, several hours to propagate.

## 4. Create the Ubuntu VM in Hyper-V

### 4.1 Recommended resources

The currently allocated resources are suitable:

- 4 virtual CPU cores
- Approximately 6.6 GB RAM
- 150 GB storage
- Network connection above 1 Gbps
- Ubuntu Server 22.04 LTS

Use a **Generation 2** VM when possible.

### 4.2 Install Ubuntu

During Ubuntu installation:

1. Select Ubuntu Server 22.04 LTS.
2. Use the entire virtual disk unless another partition layout is required.
3. Create an administrative user. This guide uses `meeting` as an example.
4. Install OpenSSH Server when prompted.
5. Do not install unnecessary optional server packages.
6. Complete the installation and reboot.

Initially, the VM can use Hyper-V's Default Switch so Ubuntu can download updates. It will later move to a dedicated static NAT network.

## 5. Update Ubuntu

Log in to Ubuntu and run:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl openssl ufw nano
```

If Ubuntu displays:

```text
Could not get lock /var/lib/dpkg/lock-frontend
Held by process ... unattended-upgr
```

wait for the automatic updater to finish. Do not delete the lock file.

Check whether a reboot is required:

```bash
test -f /var/run/reboot-required && echo "Reboot required" || echo "No reboot required"
```

If required:

```bash
sudo reboot
```

Set the timezone:

```bash
sudo timedatectl set-timezone America/New_York
timedatectl
```

## 6. Create the dedicated Hyper-V NAT network

Do these commands on the **Windows host**, not inside Ubuntu.

Open **PowerShell as Administrator**.

### 6.1 Check whether this guide was already applied

```powershell
Get-VMSwitch -Name "LiveKitSwitch" -ErrorAction SilentlyContinue
Get-NetNat -Name "LiveKitNAT" -ErrorAction SilentlyContinue
```

If both already exist and are correctly configured, do not create duplicates.

### 6.2 Create the internal switch

```powershell
New-VMSwitch -SwitchName "LiveKitSwitch" -SwitchType Internal
```

Confirm its Windows adapter:

```powershell
Get-NetAdapter | Where-Object Name -Like "*LiveKitSwitch*"
```

### 6.3 Give Windows the gateway address

```powershell
New-NetIPAddress `
  -InterfaceAlias "vEthernet (LiveKitSwitch)" `
  -IPAddress 172.30.0.1 `
  -PrefixLength 24
```

If PowerShell says the address already exists, inspect it instead of adding it again:

```powershell
Get-NetIPAddress -InterfaceAlias "vEthernet (LiveKitSwitch)"
```

### 6.4 Create Windows NAT

```powershell
New-NetNat `
  -Name "LiveKitNAT" `
  -InternalIPInterfaceAddressPrefix "172.30.0.0/24"
```

Verify:

```powershell
Get-NetNat -Name "LiveKitNAT"
```

## 7. Attach Ubuntu to the new switch

Inside Ubuntu:

```bash
sudo poweroff
```

In Hyper-V Manager:

1. Right-click the Ubuntu VM.
2. Select **Settings**.
3. Select **Network Adapter**.
4. Change **Virtual switch** to `LiveKitSwitch`.
5. Select **Apply**.
6. Select **OK**.
7. Start the VM.

The VM may initially have no internet. Configure its static address through the Hyper-V console.

## 8. Configure Ubuntu's permanent IP

Log in through the Hyper-V console.

Identify the interface:

```bash
ip link
```

This guide assumes it is `eth0`, as shown in the original VM screenshot. If it has a different name, substitute that name below.

List Netplan files:

```bash
ls -l /etc/netplan
```

Open the existing YAML file. Common names include:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

or:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Use the filename that actually exists. Replace its network configuration with:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 172.30.0.10/24
      routes:
        - to: default
          via: 172.30.0.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

YAML requires spaces, not tabs.

In Nano:

1. Press `Ctrl+O`.
2. Press Enter.
3. Press `Ctrl+X`.

Secure and validate the file:

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan generate
```

If `netplan generate` reports no error:

```bash
sudo netplan apply
```

Verify:

```bash
ip addr show eth0
ip route
```

Expected address:

```text
172.30.0.10/24
```

Expected route:

```text
default via 172.30.0.1
```

Test the Windows gateway:

```bash
ping -c 4 172.30.0.1
```

Test internet routing:

```bash
ping -c 4 1.1.1.1
```

Test DNS:

```bash
ping -c 4 google.com
```

All three must work before continuing.

From Windows, test the VM:

```powershell
ping 172.30.0.10
```

Test SSH:

```powershell
ssh meeting@172.30.0.10
```

Replace `meeting` with the actual Ubuntu username.

## 9. Configure Windows public port forwarding

Run these commands in **Administrator PowerShell** on Windows.

### 9.1 Check existing mappings

```powershell
Get-NetNatStaticMapping -NatName "LiveKitNAT" -ErrorAction SilentlyContinue
```

Do not add a duplicate mapping for the same protocol and public port.

### 9.2 Add mappings

HTTP for certificate issuance:

```powershell
Add-NetNatStaticMapping `
  -NatName "LiveKitNAT" `
  -Protocol TCP `
  -ExternalIPAddress "0.0.0.0/0" `
  -ExternalPort 80 `
  -InternalIPAddress 172.30.0.10 `
  -InternalPort 80
```

HTTPS and secure WebSocket signaling:

```powershell
Add-NetNatStaticMapping `
  -NatName "LiveKitNAT" `
  -Protocol TCP `
  -ExternalIPAddress "0.0.0.0/0" `
  -ExternalPort 443 `
  -InternalIPAddress 172.30.0.10 `
  -InternalPort 443
```

WebRTC TCP fallback:

```powershell
Add-NetNatStaticMapping `
  -NatName "LiveKitNAT" `
  -Protocol TCP `
  -ExternalIPAddress "0.0.0.0/0" `
  -ExternalPort 7881 `
  -InternalIPAddress 172.30.0.10 `
  -InternalPort 7881
```

WebRTC UDP mux:

```powershell
Add-NetNatStaticMapping `
  -NatName "LiveKitNAT" `
  -Protocol UDP `
  -ExternalIPAddress "0.0.0.0/0" `
  -ExternalPort 7882 `
  -InternalIPAddress 172.30.0.10 `
  -InternalPort 7882
```

TURN/STUN UDP:

```powershell
Add-NetNatStaticMapping `
  -NatName "LiveKitNAT" `
  -Protocol UDP `
  -ExternalIPAddress "0.0.0.0/0" `
  -ExternalPort 3478 `
  -InternalIPAddress 172.30.0.10 `
  -InternalPort 3478
```

Verify all mappings:

```powershell
Get-NetNatStaticMapping -NatName "LiveKitNAT" |
  Sort-Object Protocol,ExternalPort |
  Format-Table -AutoSize
```

## 10. Configure Windows Firewall

In Administrator PowerShell:

```powershell
New-NetFirewallRule `
  -DisplayName "LiveKit TCP Inbound" `
  -Direction Inbound `
  -Action Allow `
  -Protocol TCP `
  -LocalPort 80,443,7881
```

```powershell
New-NetFirewallRule `
  -DisplayName "LiveKit UDP Inbound" `
  -Direction Inbound `
  -Action Allow `
  -Protocol UDP `
  -LocalPort 3478,7882
```

Verify:

```powershell
Get-NetFirewallRule -DisplayName "LiveKit*" |
  Format-Table DisplayName,Enabled,Direction,Action
```

Do not publicly forward SSH unless remote SSH is truly required. Administrators can connect from Windows to `172.30.0.10`.

## 11. Configure Ubuntu Firewall

Inside Ubuntu, allow SSH from only the private Hyper-V subnet:

```bash
sudo ufw allow from 172.30.0.0/24 to any port 22 proto tcp
```

Allow LiveKit traffic:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 7881/tcp
sudo ufw allow 7882/udp
sudo ufw allow 3478/udp
```

Enable and inspect UFW:

```bash
sudo ufw enable
sudo ufw status verbose
```

Keep the Hyper-V console open until SSH has been tested successfully.

## 12. Install Docker

Inside Ubuntu:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker "$USER"
```

Log out and sign back in, or reboot:

```bash
sudo reboot
```

Verify:

```bash
docker --version
docker compose version
sudo systemctl status docker --no-pager
```

Docker should report `active (running)`.

## 13. Generate LiveKit credentials

Generate a key and secret inside Ubuntu:

```bash
LIVEKIT_KEY="API$(openssl rand -hex 8)"
LIVEKIT_SECRET="$(openssl rand -hex 32)"

echo "LiveKit API key: $LIVEKIT_KEY"
echo "LiveKit API secret: $LIVEKIT_SECRET"
```

Copy both values into a secure password manager.

Important:

- Do not post the secret in chat.
- Do not put the secret in browser JavaScript.
- Do not commit it to a public Git repository.
- The portal backend and LiveKit server must use the same key/secret pair.

The shell variables disappear after logout. Saving them securely is essential.

## 14. Create the LiveKit deployment directory

```bash
sudo mkdir -p /opt/livekit
sudo chown "$USER":"$USER" /opt/livekit
cd /opt/livekit
```

## 15. Create `livekit.yaml`

Open the file:

```bash
nano /opt/livekit/livekit.yaml
```

Paste:

```yaml
port: 7880
log_level: info

rtc:
  use_external_ip: true
  tcp_port: 7881
  udp_port: 7882

keys:
  YOUR_API_KEY: "YOUR_API_SECRET"

room:
  empty_timeout: 300
  departure_timeout: 20
  max_participants: 50

turn:
  enabled: true
  udp_port: 3478
```

Replace `YOUR_API_KEY` and `YOUR_API_SECRET` with the credentials generated in the previous section.

Example structure only:

```yaml
keys:
  API_EXAMPLE_ONLY: "EXAMPLE_SECRET_ONLY"
```

Do not configure `rtc.port_range_start` or `rtc.port_range_end` when using `rtc.udp_port`.

Save with `Ctrl+O`, Enter, and `Ctrl+X`.

Protect the file:

```bash
chmod 600 /opt/livekit/livekit.yaml
```

## 16. Create the Caddy configuration

Open:

```bash
nano /opt/livekit/Caddyfile
```

Paste, replacing the domain:

```caddy
YOUR_LIVEKIT_DOMAIN {
    reverse_proxy 127.0.0.1:7880
}
```

Example:

```caddy
meet.mbrnetworking.org {
    reverse_proxy 127.0.0.1:7880
}
```

Caddy will request and renew a trusted TLS certificate automatically. It can only succeed if:

- DNS points to `15.235.87.111`.
- Windows forwards TCP 80 and 443 to the VM.
- Windows and Ubuntu firewalls allow TCP 80 and 443.
- No other service occupies those ports.

## 17. Create `docker-compose.yaml`

Open:

```bash
nano /opt/livekit/docker-compose.yaml
```

Paste:

```yaml
services:
  livekit:
    image: livekit/livekit-server:latest
    container_name: livekit
    command: --config /etc/livekit.yaml
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml:ro

  caddy:
    image: caddy:2
    container_name: livekit-caddy
    network_mode: host
    restart: unless-stopped
    depends_on:
      - livekit
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

Validate the Compose file:

```bash
cd /opt/livekit
docker compose config
```

Fix any YAML error before continuing.

## 18. Start LiveKit and Caddy

```bash
cd /opt/livekit
docker compose pull
docker compose up -d
```

Check container status:

```bash
docker compose ps
```

Both containers should show `Up` or `running`.

Inspect LiveKit logs:

```bash
docker compose logs --tail=100 livekit
```

Inspect Caddy logs:

```bash
docker compose logs --tail=100 caddy
```

Follow logs live with:

```bash
docker compose logs -f
```

Stop following with `Ctrl+C`. This does not stop the containers.

## 19. Verify listening ports in Ubuntu

```bash
sudo ss -lntup | grep -E ':80|:443|:7880|:7881|:7882|:3478'
```

Expected services include:

- TCP 80: Caddy
- TCP 443: Caddy
- TCP 7880: LiveKit internal API
- TCP 7881: LiveKit WebRTC TCP
- UDP 7882: LiveKit WebRTC UDP mux
- UDP 3478: LiveKit TURN/STUN

## 20. Test LiveKit locally

Inside Ubuntu:

```bash
curl -I http://127.0.0.1:7880
```

A `404` response is acceptable. It proves LiveKit's HTTP server is answering.

Test Caddy locally while supplying the hostname:

```bash
curl -I -H "Host: YOUR_LIVEKIT_DOMAIN" http://127.0.0.1
```

## 21. Verify TLS and external access

Run these tests from a different internet connection when possible—for example, a laptop using a mobile hotspot. Hairpin NAT tests from the Windows host may behave differently from real external traffic.

### 21.1 Test DNS

```bash
nslookup YOUR_LIVEKIT_DOMAIN
```

It must return `15.235.87.111`.

### 21.2 Test HTTPS

```bash
curl -I https://YOUR_LIVEKIT_DOMAIN
```

A trusted HTTPS connection with a `404` response is acceptable.

### 21.3 Inspect the certificate

```bash
openssl s_client \
  -connect YOUR_LIVEKIT_DOMAIN:443 \
  -servername YOUR_LIVEKIT_DOMAIN
```

Look for:

```text
Verify return code: 0 (ok)
```

### 21.4 Test Windows TCP forwarding

From an external Windows machine:

```powershell
Test-NetConnection YOUR_LIVEKIT_DOMAIN -Port 443
Test-NetConnection YOUR_LIVEKIT_DOMAIN -Port 7881
```

`TcpTestSucceeded` should be `True`.

PowerShell's `Test-NetConnection` does not provide a dependable UDP connectivity test. A real LiveKit connection test is the best verification for UDP media.

## 22. Test LiveKit before connecting the portal

Install the LiveKit CLI on a trusted development machine, or follow the current CLI installation instructions from LiveKit.

Configure a project using the self-hosted values:

```bash
lk project add mbr-self-hosted \
  --url wss://YOUR_LIVEKIT_DOMAIN \
  --api-key YOUR_API_KEY \
  --api-secret YOUR_API_SECRET \
  --default
```

Generate a test token:

```bash
lk token create \
  --join \
  --room test-room \
  --identity test-user \
  --valid-for 1h
```

Use LiveKit Meet's Custom connection option to test the server URL and token. Test from two separate browsers or devices.

Verify all of the following before touching the portal:

- Both users connect to the same room.
- Local camera preview works.
- Remote camera is visible.
- Microphone audio is audible.
- Active-speaker status changes.
- Screen sharing works.
- A device on mobile data can connect.
- A device behind a different Wi-Fi network can connect.

If signaling works but remote video/audio does not, investigate UDP `7882`, TCP `7881`, NAT, and firewalls.

## 23. Connect the MBR portal only after standalone testing

The portal will eventually need:

```text
MBR_LIVEKIT_URL=wss://YOUR_LIVEKIT_DOMAIN
MBR_LIVEKIT_API_KEY=YOUR_API_KEY
MBR_LIVEKIT_API_SECRET=YOUR_API_SECRET
```

The API secret belongs only on the PHP server. Never include it in HTML or JavaScript.

The portal Content Security Policy must permit the selected hostname in `connect-src`, for example:

```text
https://meet.mbrnetworking.org
wss://meet.mbrnetworking.org
```

Do not remove existing Google, Firebase, or self-origin entries when adding the LiveKit hostname.

After integration, the portal backend should generate tokens using the same API key and secret configured in `/opt/livekit/livekit.yaml`.

## 24. Reboot and persistence test

Reboot Ubuntu:

```bash
sudo reboot
```

After it returns:

```bash
ip addr show eth0
docker ps
sudo ss -lntup | grep -E ':80|:443|:7880|:7881|:7882|:3478'
```

Confirm:

- Ubuntu still has `172.30.0.10`.
- Docker starts automatically.
- LiveKit and Caddy containers restart automatically.
- The public domain remains reachable.

Rebooting Windows should also preserve the internal switch, NAT, static mappings, and firewall rules. Verify after a Windows maintenance reboot:

```powershell
Get-NetNat -Name "LiveKitNAT"
Get-NetNatStaticMapping -NatName "LiveKitNAT"
```

## 25. Routine administration

### View status

```bash
cd /opt/livekit
docker compose ps
docker compose logs --tail=100
```

### Restart services

```bash
cd /opt/livekit
docker compose restart
```

### Stop services

```bash
cd /opt/livekit
docker compose down
```

### Start services

```bash
cd /opt/livekit
docker compose up -d
```

### Update container images

Review LiveKit release notes before production upgrades. Then:

```bash
cd /opt/livekit
docker compose pull
docker compose up -d
docker image prune
```

Avoid blindly upgrading during an active meeting period.

## 26. Backups

Back up these files securely:

```text
/opt/livekit/livekit.yaml
/opt/livekit/Caddyfile
/opt/livekit/docker-compose.yaml
```

The LiveKit YAML contains the API secret and must be encrypted or stored in a secure password manager.

Back up Caddy certificate state:

```bash
cd /opt/livekit
docker run --rm \
  -v livekit_caddy_data:/source:ro \
  -v "$PWD":/backup \
  alpine tar czf /backup/caddy-data-backup.tar.gz -C /source .
```

The exact Docker volume prefix may vary. Confirm with:

```bash
docker volume ls
```

## 27. Security checklist

- Keep Ubuntu updated.
- Do not expose port `7880` directly to the internet.
- Restrict SSH to the internal subnet or trusted administrative IPs.
- Use long, randomly generated API secrets.
- Never store the API secret in frontend code.
- Rotate credentials if they are accidentally disclosed.
- Keep Windows Defender Firewall enabled.
- Keep Ubuntu UFW enabled.
- Back up configurations before upgrades.
- Review LiveKit and Caddy logs periodically.
- Use a trusted TLS certificate; browsers will not accept self-signed certificates for production WebRTC.

## 28. Troubleshooting

### Ubuntu has no internet after changing switches

Check:

```bash
ip addr show eth0
ip route
ping -c 4 172.30.0.1
ping -c 4 1.1.1.1
```

On Windows:

```powershell
Get-NetNat -Name "LiveKitNAT"
Get-NetIPAddress -InterfaceAlias "vEthernet (LiveKitSwitch)"
```

Expected gateway address: `172.30.0.1/24`.

### Netplan reports a YAML error

YAML indentation is significant. Use spaces, not tabs. Reopen the file:

```bash
sudo nano /etc/netplan/YOUR_FILE.yaml
sudo netplan generate
```

Do not apply until `netplan generate` succeeds.

### Caddy cannot obtain a certificate

Check:

```bash
cd /opt/livekit
docker compose logs --tail=200 caddy
```

Confirm:

- DNS resolves to `15.235.87.111`.
- TCP 80 and 443 are forwarded.
- Firewalls allow TCP 80 and 443.
- Another Windows process is not using those ports.
- Cloudflare proxy is disabled during setup.

### HTTPS works but meetings have no video/audio

This usually means signaling works but RTC media does not.

Check Ubuntu listeners:

```bash
sudo ss -lntup | grep -E ':7881|:7882|:3478'
```

Check Windows mappings:

```powershell
Get-NetNatStaticMapping -NatName "LiveKitNAT"
```

Required media mappings:

- TCP `7881`
- UDP `7882`
- UDP `3478`

### Container exits or restarts

```bash
cd /opt/livekit
docker compose ps
docker compose logs --tail=200 livekit
```

Common causes include:

- Invalid YAML
- API secret formatting error
- Port already in use
- Invalid configuration key
- Insufficient permissions reading `livekit.yaml`

### Check which process uses a Linux port

```bash
sudo ss -lntup
```

### Check which process uses a Windows port

```powershell
Get-NetTCPConnection -State Listen |
  Where-Object LocalPort -In 80,443,7881 |
  Select-Object LocalAddress,LocalPort,OwningProcess
```

Then:

```powershell
Get-Process -Id THE_PROCESS_ID
```

## 29. Important limitations of this first deployment

This is a single-server deployment. If the Windows host, Ubuntu VM, Docker, or internet connection fails, meetings stop.

It does not yet provide:

- Multi-region redundancy
- A second LiveKit node
- External Redis clustering
- Automatic failover
- Recording/egress
- RTMP ingress
- SIP telephony
- TURN/TLS on TCP 443 for the most restrictive corporate firewalls

The enabled TURN/UDP service improves connectivity, while RTC/TCP `7881` provides a TCP fallback. TURN/TLS can be added later after the basic server is stable, but sharing TCP 443 behind one Windows public IP requires additional SNI-aware routing or a separate public IP.

## 30. Official references

- LiveKit VM deployment: <https://docs.livekit.io/transport/self-hosting/vm/>
- LiveKit ports and firewall: <https://docs.livekit.io/transport/self-hosting/ports-firewall/>
- LiveKit deployment overview: <https://docs.livekit.io/transport/self-hosting/deployment/>
- LiveKit self-hosting overview: <https://docs.livekit.io/transport/self-hosting/>

Use the official documentation to verify configuration changes when upgrading to a newer LiveKit release.

