---
title: Home Server Setup Notes
---

* TOC
{:toc}

## Key preparations

- internal network is `enp2s0`
- external network is `eno1`

I want to have the external network set up for internet access only, and the internal network for LAN access only. The server should not route traffic between the two networks. Effectively, the server acts as a client on both networks, but does not forward traffic between them, a reverse proxy appliance, and VPN endpoint.

## Disable IP forwarding

check that IP forwarding isn't enabled:

```bash
sudo sysctl net.ipv4.ip_forward
```

it returned

`net.ipv4.ip_forward = 1`

created a new file

```bash
sudo nano /etc/sysctl.d/99-network-hardening.conf
```

and added

```code
# Disable IPv4 packet forwarding (no gateway duties)
net.ipv4.ip_forward = 0

# Disable IPv6 forwarding unless explicitly needed
net.ipv6.conf.all.forwarding = 0

# Ignore ICMP broadcast requests (mitigate smurf attacks)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Do not accept source-routed packets
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Do not accept ICMP redirects (prevent malicious route injection)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Do not send ICMP redirects (stay quiet as a router)
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Enable SYN cookies (protect against SYN flood DoS)
net.ipv4.tcp_syncookies = 1

# Log suspicious packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Reverse path filtering (helps prevent IP spoofing)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
```

reload

```bash
sudo sysctl --system
```

## Configure interfaces separately

using systemd networkd

```bash
sudo vim /etc/systemd/network/20-enp2s0-lan.network
```


```ini
[Match]
Name=enp2s0

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=2000
```

```bash
sudo vim /etc/systemd/network/10-eno1-external.network
```

```ini
[Match]
Name=eno1

[Network]
DHCP=yes
DNS=1.1.1.1 8.8.8.8   # optional: override ISP DNS with public resolvers

[DHCPv4]
RouteMetric=10       # Low metric = high priority - ensures eno1's default route wins
UseRoutes=yes
```

enable and start:

```bash
sudo systemctl enable --now systemd-networkd
sudo systemctl restart systemd-networkd
```

**NOTE**: If `systemctl restart systemd-networkd` hangs and doesn't return, you may need to reboot the server. This can happen when changing DHCP settings. To avoid SSH lockout in the future:

```bash
# Check config syntax before applying
sudo networkctl reload
# Or test with a timeout that auto-reverts
sudo systemctl restart systemd-networkd & sleep 10 && ip route show
```

After reboot, verify the routes:

```bash
ip route show
# Should only see default via public IP address (eno1), NOT via 192.168.0.1
```

works!

## Firewall with iptables

Since Docker doesn't work properly with nftables, use iptables-legacy instead.

### Install and configure iptables

Install iptables-legacy:

```bash
sudo apt update
sudo apt install iptables iptables-persistent
```

Switch to iptables-legacy:

Debian provides two “flavors” of iptables binaries: iptables-nft and iptables-legacy. You can select which one is active:

```bash
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

### Disable nftables

**IMPORTANT**: If nftables is installed, it must be completely disabled:

```bash
# Stop and disable nftables service
sudo systemctl stop nftables
sudo systemctl disable nftables

# Clear any active nftables rules
sudo nft flush ruleset
```

You can optionally delete `/etc/nftables.conf` or rename it to `.bak` for reference. Since the service is disabled, the file won't be loaded on boot anyway.

This makes it so that:

- Docker manages firewall rules via iptables-legacy
- eno1: Docker/NPM handles 80/443 from internet
- enp2s0: LAN access to NPM admin UI (port 81) works
- No forwarding: box won't act as a router
- Minimal attack surface

## Set up Nginx Proxy Manager:

(install Docker and Docker Compose)

### Create directories for NPM

```bash
sudo mkdir -p /opt/npm/data /opt/npm/letsencrypt
sudo chown -R $USER:$USER /opt/npm
cd /opt/npm
```

### Create NPM docker-compose.yml

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped

    ports:
      - '80:80'              # Public HTTP Port
      - '443:443'            # Public HTTPS Port
      - '192.168.0.101:81:81'  # Admin Web Port (LAN only)

    # Use host's DNS for proper LAN service resolution
    dns:
      - 192.168.0.2          # LAN DNS
      - 1.1.1.1              # Cloudflare fallback

    environment:
      DISABLE_IPV6: 'true'
      TZ: 'America/Chicago'

    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

The `dns` configuration is critical - it allows NPM to resolve and reach internal LAN services properly while still accessing the internet.

## Default Landing Page

Set up a simple default page for `yourdomain.com` and catch-all subdomains.

### Create landing page directory

```bash
sudo mkdir -p /opt/default-page
sudo chown -R $USER:$USER /opt/default-page
cd /opt/default-page
```

### Create index.html

```bash
vim index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>yourdomain.com</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            text-align: center;
            max-width: 600px;
        }
        h1 {
            font-size: 4rem;
            margin-bottom: 1rem;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
        }
        p {
            font-size: 1.5rem;
            opacity: 0.9;
            line-height: 1.6;
        }
        .subdomain {
            margin-top: 2rem;
            padding: 1rem;
            background: rgba(255,255,255,0.1);
            border-radius: 8px;
            font-size: 1rem;
            opacity: 0.7;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>👋</h1>
        <p>Nothing to see here.</p>
        <div class="subdomain">yourdomain.com</div>
    </div>
</body>
</html>
```

### Create robots.txt (deny crawlers)

```bash
vim robots.txt
```

```text
User-agent: *
Disallow: /
```

### Create default-page docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  default-page:
    image: nginx:alpine
    container_name: default-page
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
      - ./robots.txt:/usr/share/nginx/html/robots.txt:ro
    ports:
      - "192.168.0.101:8090:80"
    restart: unless-stopped
```

### Start the service

```bash
docker compose up -d

# Check logs
docker logs -f default-page
```

### Configure NPM proxy hosts

1. Open NPM at `http://192.168.0.101:81`
2. Add two proxy hosts:

**For root domain:**
- Domain Names: `yourdomain.com`
- Scheme: `http`
- Forward Hostname/IP: `192.168.0.101`
- Forward Port: `8090`
- SSL: Request new certificate, Force SSL

**For catch-all subdomains:**
- Domain Names: `yourdomain.com` (should show up once saved with *)
- Scheme: `http`
- Forward Hostname/IP: `192.168.0.101`
- Forward Port: `8090`
- SSL: Request new certificate, Force SSL

Now any undefined subdomain will show the default page instead of an error.

## Dynamic DNS with Namecheap

To keep your domain pointing to your external IP address, set up dynamic DNS updates.

### Enable Dynamic DNS in Namecheap

1. Log in to Namecheap
2. Go to Domain List → Manage → Advanced DNS
3. Enable "Dynamic DNS" toggle
4. Note the Dynamic DNS Password (you'll need this)

### Configure DDNS with ddclient

Install ddclient:

```bash
sudo apt update
sudo apt install ddclient
```

During installation, you can skip the configuration prompts or configure for Namecheap directly.

Edit the configuration:

```bash
sudo vim /etc/ddclient.conf
```

Replace contents with:

```ini
# Configuration for Namecheap
daemon=600                   # Check every 10 minutes
syslog=yes                   # Log via syslog
pid=/var/run/ddclient.pid    # PID file location
ssl=yes                      # Use SSL

# Get IP from interface
use=if, if=eno1

# Namecheap settings
protocol=namecheap
server=dynamicdns.park-your-domain.com
login=yourdomain.com
password='your-dynamic-dns-password'
@,*
```

**Note**: 

- Replace `your-dynamic-dns-password` with the password from Namecheap's Dynamic DNS settings
- The `@,*` syntax updates both the root domain (@.yourdomain.com) and wildcard (*.yourdomain.com)
- Make sure both @ and * records are enabled for Dynamic DNS in Namecheap

Restart and enable:

```bash
sudo systemctl restart ddclient
sudo systemctl enable ddclient
sudo systemctl status ddclient

# Check the logs
sudo journalctl -u ddclient -f
```

Test the update:

```bash
sudo ddclient -daemon=0 -debug -verbose -noquiet
```

## WireGuard VPN with wg-easy

Set up WireGuard VPN with a web UI for easy client management.

### Create WireGuard directory

```bash
sudo mkdir -p /opt/wireguard
sudo chown -R $USER:$USER /opt/wireguard
cd /opt/wireguard
```

### Create WireGuard docker-compose.yml

First, generate a password hash:

```bash
docker run -it ghcr.io/wg-easy/wg-easy wgpw 'YourSecurePasswordHere'
```

This will output a bcrypt hash. Copy it for the next step.

```bash
vim docker-compose.yml
```

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    environment:
      - LANG=en
      - WG_HOST=yourdomain.com          # Your dynamic DNS domain
      - PASSWORD_HASH=$$2a$$12$$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  # Paste your hash here (escape $ with $$)
      - PORT=51821                   # Web UI port
      - WG_PORT=51820                # WireGuard VPN port
      - WG_DEFAULT_ADDRESS=10.13.13.x
      - WG_DEFAULT_DNS=192.168.0.2   # Your LAN DNS
      - WG_ALLOWED_IPS=192.168.0.0/24, 10.13.13.0/24  # LAN + VPN subnet
      - WG_PERSISTENT_KEEPALIVE=25
      - WG_MTU=1420
    volumes:
      - /opt/wireguard:/etc/wireguard
    ports:
      - "51820:51820/udp"            # WireGuard port
      - "192.168.0.101:51821:51821/tcp"  # Web UI (LAN only)
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
    restart: unless-stopped
```

**Important**: In docker-compose.yml, escape each `$` in the hash by doubling it (`$$`).

### Start WireGuard

```bash
docker compose up -d

# Check logs
docker logs -f wg-easy
```

Docker will automatically handle the necessary iptables rules for WireGuard.

### Access Web UI

Open `http://192.168.0.101:51821` from your LAN and log in with the password you set.

From the web interface you can:
- Add/remove clients
- Generate QR codes
- Download config files
- See connected clients
- Enable/disable clients

### Access client configurations

WireGuard will generate client configs with QR codes in the logs. You can also find them in:

```bash
ls /opt/wireguard/config/peer*
```

Each peer directory contains:
- `peer1.conf` - Configuration file
- `peer1.png` - QR code for mobile apps

### Port forwarding

In your router, forward UDP port 51820 to 192.168.0.101 (this server).

### Connect clients

**Mobile (iOS/Android):**
1. Install WireGuard app
2. Scan QR code from docker logs or `/opt/wireguard/config/peer1/peer1.png`

**Desktop:**
1. Install WireGuard client
2. Import `/opt/wireguard/config/peer1/peer1.conf`

Once connected, you'll have access to your entire LAN (192.168.0.0/24) through the VPN.

### Check connected peers

```bash
docker exec wg-easy wg show
```

### Expose WireGuard Web UI via NPM (Optional)

For remote management of WireGuard clients, expose the web UI through NPM:

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `wireguard.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `51821`
   - Enable "Websockets Support"
4. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Accept Let's Encrypt Terms
5. Save

Now you can manage WireGuard clients remotely at `https://wireguard.yourdomain.com` using your password.

**Security note**: The web UI is already protected with bcrypt password hashing. Make sure you used a strong password when generating the hash.

## External USB Drive as Samba Share Storage

Mount an external USB drive at `/srv/share` for Samba shares and Plex library.

### Identify and format the drive

Plug in the USB drive and identify it:

```bash
lsblk
# Should show sdb as 14T
```

Format the drive with ext4:

```bash
sudo mkfs.ext4 /dev/sdb1
```

### Mount the drive

Create mount point and get UUID:

```bash
sudo mkdir -p /srv/share
sudo blkid /dev/sdb1
```

Add to `/etc/fstab` for automatic mounting:

```bash
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /srv/share ext4 defaults 0 2
```

Mount and set permissions:

```bash
sudo mount -a
sudo chown -R $USER:$USER /srv/share
```

Verify it's mounted:

```bash
df -h | grep /srv/share
```

### Install and configure Samba

Install Samba to share `/srv/share`:

```bash
sudo apt install samba
```

Edit Samba config:

```bash
sudo vim /etc/samba/smb.conf
```

Add the following share definition at the end of the file:

```ini
[share]
   path = /srv/share
   browseable = yes
   writable = yes
   guest ok = no
```

### Create Samba user

Create Samba user:

```bash
sudo smbpasswd -a $USER
```

### Configure firewall

Adjust firewall to allow Samba:

```bash
# Allow Samba on LAN
sudo iptables -A INPUT -i enp2s0 -p udp -m multiport --dports 137,138 -j ACCEPT
sudo iptables -A INPUT -i enp2s0 -p tcp -m multiport --dports 139,445 -j ACCEPT

# Block Samba externally
sudo iptables -A INPUT -i eno1 -p udp -m multiport --dports 137,138 -j DROP
sudo iptables -A INPUT -i eno1 -p tcp -m multiport --dports 139,445 -j DROP
```

Make it persistent across reboots:

```bash
sudo netfilter-persistent save
```

## Plex Media Server

Restore Plex Media Server with your existing library at `/srv/share/Library`.

### Create Plex directory structure

```bash
sudo mkdir -p /opt/plex/config
sudo chown -R $USER:$USER /opt/plex
cd /opt/plex
```

### Create docker-compose.yml for Plex

```bash
vim docker-compose.yml
```

```yaml
services:
  plex:
    image: plexinc/pms-docker:latest
    container_name: plex
    network_mode: host
    environment:
      - TZ=America/Chicago
      - PLEX_UID=1000
      - PLEX_GID=1000
    volumes:
      - /opt/plex/config:/config
      - /srv/share/Library:/data
    devices:
      - /dev/dri:/dev/dri  # Intel QuickSync hardware transcoding (i5-8500T)
    restart: unless-stopped
```

**Note:**

- Using `network_mode: host` allows Plex to properly discover clients on your LAN and makes DLNA work correctly
- The `/dev/dri` device passthrough enables Intel QuickSync hardware transcoding on the i5-8500T's UHD Graphics 630
- After starting Plex, enable hardware acceleration in Settings → Transcoder

### Start Plex

```bash
docker compose up -d

# Check logs
docker logs -f plex
```

### Allow Plex through firewall

Since Plex uses `host` networking mode, Docker doesn't automatically manage firewall rules. Add an iptables rule to allow access from the LAN:

```bash
sudo iptables -I INPUT -i enp2s0 -p tcp --dport 32400 -j ACCEPT
```

Make it persistent across reboots:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

### Access Plex

Open `http://192.168.0.101:32400/web` from your LAN to set up Plex.

During setup:

1. Sign in with your Plex account
2. Name your server
3. Add library pointing to `/data` (which maps to `/srv/share/Library`)
4. Plex should recognize your existing library structure

### Enable Remote Access

Plex supports two methods for remote access: direct connection (better performance) and reverse proxy (uses existing SSL setup).

#### Method 1: Direct Connection (Recommended for streaming)

Allow Plex through the external interface:

```bash
sudo iptables -I INPUT -i eno1 -p tcp --dport 32400 -j ACCEPT
sudo netfilter-persistent save
```

Forward port 32400 in your router: External port 32400 → 192.168.0.101:32400

In Plex Settings:

1. Go to Settings → Remote Access
2. Click "Enable Remote Access"
3. Manually specify port: `32400`

#### Method 2: Reverse Proxy via NPM

Add Plex to Nginx Proxy Manager for SSL access:

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `plex.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `32400`
   - Enable "Websockets Support"
4. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Accept Let's Encrypt Terms
5. Save

In Plex Settings → Network:

- Add `https://plex.yourdomain.com:443` to "Custom server access URLs"

**Note**: Both methods can be used simultaneously. Direct connection will be preferred for local network and external streaming, while the reverse proxy provides additional SSL access.

## ntfy - Push Notification Service

Set up ntfy for simple push notifications to your devices.

### Create ntfy directory

```bash
sudo mkdir -p /opt/ntfy
sudo chown -R $USER:$USER /opt/ntfy
cd /opt/ntfy
```

### Create ntfy docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  ntfy:
    image: binwiederhier/ntfy:latest
    container_name: ntfy
    command:
      - serve
    environment:
      - TZ=America/Chicago
    volumes:
      - /opt/ntfy/cache:/var/cache/ntfy
      - /opt/ntfy/etc:/etc/ntfy
    ports:
      - "192.168.0.101:8080:80"  # Web UI and API (LAN only)
    restart: unless-stopped
```

### Start ntfy

```bash
docker compose up -d

# Check logs
docker logs -f ntfy
```

### Access ntfy

Open `http://192.168.0.101:8080` from your LAN.

### Send a test notification

```bash
curl -d "Hello from ntfy!" http://192.168.0.101:8080/test
```

Then subscribe to the `test` topic in the web UI or mobile app to see the notification.

### Mobile apps

- **Android**: Install from Play Store or F-Droid
- **iOS**: Install from App Store

In the app, add your server: `http://192.168.0.101:8080`

### Expose ntfy via NPM

For external access, add ntfy to Nginx Proxy Manager:

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `ntfy.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `8080`
   - Enable "Websockets Support"
4. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Accept Let's Encrypt Terms
5. Save

Then you can send notifications from anywhere:

```bash
curl -d "External notification" https://ntfy.yourdomain.com/alerts
```

Update your mobile apps to use `https://ntfy.yourdomain.com` instead of the local IP.

## Nextcloud with Dedicated Drive

Set up Nextcloud with a dedicated 1.9TB drive for data storage.

### Format and mount the drive

First, identify the drive:

```bash
lsblk
# Should show sda as 1.9T
```

Format the drive with ext4:

```bash
sudo mkfs.ext4 /dev/sda
```

Create mount point and get UUID:

```bash
sudo mkdir -p /srv/nextcloud
sudo blkid /dev/sda
# Note the UUID value
```

Add to `/etc/fstab` for automatic mounting:

```bash
sudo vim /etc/fstab
```

Add this line (replace YOUR-UUID with the actual UUID):

```fstab
UUID=YOUR-UUID  /srv/nextcloud  ext4  defaults,nofail  0  2
```

Mount and set permissions:

```bash
sudo mount -a
sudo systemctl daemon-reload  # Reload systemd if you see the fstab modified hint
sudo chown -R 33:33 /srv/nextcloud  # www-data user for Nextcloud
```

Verify it's mounted:

```bash
df -h | grep nextcloud
```

### Give your user read access

Add your user to the www-data group to access the Nextcloud drive:

```bash
sudo usermod -a -G www-data $USER
```

Then log out and back in, or run:

```bash
newgrp www-data
```

This allows you to read files on the Nextcloud drive. The `force user/group` settings in the Samba configuration ensure any files created via SMB are properly owned by `www-data` for Nextcloud to use.

### Create Nextcloud directory

```bash
sudo mkdir -p /opt/nextcloud
sudo chown -R $USER:$USER /opt/nextcloud
cd /opt/nextcloud
```

### Create Nextcloud docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  db:
    image: mariadb:latest
    container_name: nextcloud-db
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    restart: unless-stopped
    volumes:
      - /opt/nextcloud/db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=SecureRootPassword123
      - MYSQL_PASSWORD=SecurePassword123
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - TZ=America/Chicago

  app:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - "192.168.0.101:8081:80"  # Web interface (LAN only)
    links:
      - db
    volumes:
      - /opt/nextcloud/html:/var/www/html
      - /srv/nextcloud:/var/www/html/data
    environment:
      - MYSQL_PASSWORD=SecurePassword123
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
      - TZ=America/Chicago
    depends_on:
      - db
```

**Important**: Change the passwords to secure values before starting.

(Option, just run `openssl rand -base64 32` to generate secure passwords)

### Start Nextcloud

```bash
docker compose up -d

# Watch the initialization (takes a few minutes)
docker logs -f nextcloud
```

### Access Nextcloud

Open `http://192.168.0.101:8081` from your LAN.

During first setup:

1. Create admin account
2. Data folder should already be set to `/var/www/html/data` (the 1.9TB drive)
3. Database settings are already configured via environment variables
4. Click "Install"

### Expose Nextcloud via NPM

Add Nextcloud to Nginx Proxy Manager for external access:

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `nextcloud.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `8081`
   - Enable "Websockets Support"
   - Scroll down to **Advanced** section
   - In "Custom Nginx Configuration" box, add:

     ```nginx
     client_max_body_size 10G;
     proxy_request_buffering off;
     ```

4. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Enable "HTTP/2 Support"
   - Accept Let's Encrypt Terms
5. Save

### Configure trusted domains

After setting up the reverse proxy, add the domain to Nextcloud's trusted domains.

Edit the config file on the host:

```bash
sudo vim /opt/nextcloud/html/config/config.php
```

Find the `trusted_domains` array and add your domain, plus the `overwriteprotocol` setting:

```php
'trusted_domains' =>
array (
  0 => '192.168.0.101:8081',
  1 => 'nextcloud.yourdomain.com',
),
'overwriteprotocol' => 'https',
```

The `overwriteprotocol` setting tells Nextcloud to generate HTTPS URLs even though NPM forwards to it via HTTP internally.

Restart Nextcloud:

```bash
cd /opt/nextcloud
docker restart nextcloud
```

Now you can access Nextcloud at `https://nextcloud.yourdomain.com` with all your data stored on the 1.9TB drive.

### Add Samba share for direct file access

For native file system access to Nextcloud data, add another share to the existing Samba configuration.

Edit the Samba config:

```bash
sudo vim /etc/samba/smb.conf
```

Add the Nextcloud share definition at the end of the file:

```ini
[nextcloud]
path = /srv/nextcloud
browseable = yes
writable = yes
guest ok = no
valid users = nextcloud
force user = www-data
force group = www-data
create mask = 0664
directory mask = 0775
```

Create a system user and Samba user for Nextcloud access:

```bash
# Create a system user (no login, no home directory)
sudo useradd -r -s /usr/sbin/nologin -M nextcloud

# Create Samba password for the nextcloud user
sudo smbpasswd -a nextcloud
```

**Important**: Set a secure password when prompted.

Restart Samba to apply changes:

```bash
sudo systemctl restart smbd
```

### Access the share

**Windows:**

- Open File Explorer
- Type in address bar: `\\192.168.0.101\nextcloud`
- Username: `nextcloud`
- Password: (the one you set)

**Mac:**

- Finder → Go → Connect to Server
- Server: `smb://192.168.0.101/nextcloud`
- Username: `nextcloud`
- Password: (the one you set)

**Linux:**

```bash
sudo apt install cifs-utils
sudo mount -t cifs //192.168.0.101/nextcloud /mnt/share -o username=nextcloud,uid=1000,gid=1000
```

### Sync changes to Nextcloud

After making changes via SMB, tell Nextcloud to rescan for new files:

```bash
docker exec -u www-data nextcloud php occ files:scan --all
```

Or for a specific user:

```bash
docker exec -u www-data nextcloud php occ files:scan username
```

**Note**: Files added/modified via SMB won't immediately appear in Nextcloud web interface until a scan is run. Consider setting up a cron job for periodic scans if you frequently use SMB.

## Wallabag - Read It Later Service

Self-hosted application for saving web pages to read later (like Pocket or Instapaper).

### Create Wallabag directory

```bash
sudo mkdir -p /opt/wallabag
sudo chown -R $USER:$USER /opt/wallabag
cd /opt/wallabag
```

### Create Wallabag docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  wallabag:
    image: wallabag/wallabag
    container_name: wallabag
    environment:
      - SYMFONY__ENV__DATABASE_DRIVER=pdo_sqlite
      - SYMFONY__ENV__DATABASE_NAME=wallabag
      - SYMFONY__ENV__DOMAIN_NAME=https://wallabag.yourdomain.com
      - SYMFONY__ENV__SERVER_NAME="My Wallabag"
    ports:
      - "192.168.0.101:8082:80"
    volumes:
      - ./data:/var/www/wallabag/data
      - ./images:/var/www/wallabag/web/assets/images
    restart: unless-stopped
```

**Important**: Replace `https://wallabag.yourdomain.com` with your actual domain.

**Note**: Using SQLite keeps everything simple—just one container, and the database is a single file in `./data`. Perfect for personal use.

### Start Wallabag

```bash
docker compose up -d

# Check logs
docker logs -f wallabag
```

### Allow Wallabag through firewall

If you haven't run the updated firewall script above (which now includes port 8082), add this rule:

```bash
sudo iptables -A INPUT -i enp2s0 -p tcp --dport 8082 -j ACCEPT
sudo netfilter-persistent save
```

### Expose Wallabag via NPM

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `wallabag.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `8082`
   - Enable "Websockets Support"
4. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Accept Let's Encrypt Terms
5. Save

### Access Wallabag

Open `https://wallabag.yourdomain.com`.

**Default credentials:**
- Username: `wallabag`
- Password: `wallabag`

**Security note**: Change the password immediately after logging in.

## RustDesk - Self-Hosted Remote Desktop

Set up RustDesk server for secure, self-hosted remote desktop access.

For more details, see the [official RustDesk server documentation](https://rustdesk.com/docs/en/self-host/).

### Create RustDesk directory

```bash
sudo mkdir -p /opt/rustdesk
sudo chown -R $USER:$USER /opt/rustdesk
cd /opt/rustdesk
```

### Create RustDesk docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  hbbs:
    container_name: rustdesk-hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r rustdesk.yourdomain.com:21117
    volumes:
      - ./data:/root
    network_mode: host
    restart: unless-stopped

  hbbr:
    container_name: rustdesk-hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    network_mode: host
    restart: unless-stopped
```

**Note**: Replace `rustdesk.yourdomain.com` with your actual domain.

### Start RustDesk

```bash
docker compose up -d

# Check logs
docker logs -f rustdesk-hbbs
docker logs -f rustdesk-hbbr
```

### Get the public key

After starting, retrieve the public key needed for client configuration:

```bash
cat /opt/rustdesk/data/id_ed25519.pub
```

Save this key - you'll need it when configuring RustDesk clients.

### Configure RustDesk clients

On each device you want to control or connect from:

1. Download and install RustDesk from [https://rustdesk.com](https://rustdesk.com)
2. Open RustDesk settings
3. Click "Network" → "ID Server"
4. Set ID Server to: `rustdesk.yourdomain.com`
5. Set Relay Server to: `rustdesk.yourdomain.com`
6. Click "Key" and paste the public key from earlier
7. Click "Apply"

### Usage

**To allow someone to connect to you:**

1. Open RustDesk
2. Share your ID and password with the remote user
3. They enter your ID and password in their RustDesk client

**To connect to another device:**

1. Open RustDesk
2. Enter the remote device's ID
3. Click "Connect"
4. Enter the password when prompted

## Immich - Self-Hosted Photo & Video Backup

High performance self-hosted photo and video management solution.

### Create Immich directory

```bash
sudo mkdir -p /opt/immich
sudo chown -R $USER:$USER /opt/immich
cd /opt/immich
```

### Download configuration files

Download the recommended `docker-compose.yml` and `.env` files:

```bash
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

### Configure environment

Edit the `.env` file to configure your settings:

```bash
vim .env
```

Key settings to adjust:
- `UPLOAD_LOCATION`: Set this to a path with plenty of storage. Since we have a large external drive mounted at `/srv/share`, let's use that.
- `DB_PASSWORD`: Set a secure password for the database.
- `TZ`: Set your timezone (e.g., `America/Chicago`).

Create the upload directory:

```bash
sudo mkdir -p /srv/share/Immich
sudo chown -R $USER:$USER /srv/share/Immich
```

Then update `.env`:

```ini
UPLOAD_LOCATION=/srv/share/Immich
DB_PASSWORD=YourSecurePassword
TZ=America/Chicago
```

### Start Immich

```bash
docker compose up -d

# Check logs
docker logs -f immich_server
```

### Expose Immich via NPM

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `photos.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `2283`
   - Enable "Websockets Support"
4. **Advanced tab:**
   - In "Custom Nginx Configuration" box, add:
     ```nginx
     client_max_body_size 0;
     proxy_read_timeout 600s;
     proxy_send_timeout 600s;
     send_timeout 600s;
5. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Accept Let's Encrypt Terms
6. Save

### Access Immich

Open `https://photos.yourdomain.com` or `http://192.168.0.101:2283`.

Create the admin account on first login.

### Mobile Apps

Download the Immich app for iOS or Android and connect to your server URL (`https://photos.yourdomain.com`).

### Troubleshooting: System Freeze (optional, I didn't do this)

If the system locks up during `docker compose up -d`, it's likely the **Machine Learning** container consuming all CPU resources. The i5-8500T has 6 cores, and Immich may try to use them all.

To fix this, add resource limits to `docker-compose.yml`:

```yaml
  immich-machine-learning:
    # ... existing config ...
    deploy:
      resources:
        limits:
          cpus: '4'  # Leave 2 cores for the system
          memory: 8G
```

## Calibre-Web - Ebook Library Access

Use desktop Calibre for editing/conversion; use Calibre-Web here for browsing/downloading. Sync the library from your desktop to the server; treat the server copy as read-mostly.

### Create library directory

```bash
sudo mkdir -p /srv/share/CalibreLibrary
sudo chown -R $USER:$USER /srv/share/CalibreLibrary
```

### Deploy Calibre-Web

```bash
sudo mkdir -p /opt/calibre-web
sudo chown -R $USER:$USER /opt/calibre-web
cd /opt/calibre-web
```

Create compose file:

```bash
vim docker-compose.yml
```

```yaml
services:
  calibre-web:
    image: lscr.io/linuxserver/calibre-web:latest
    container_name: calibre-web
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Chicago
      - DOCKER_MODS=linuxserver/mods:universal-calibre
    volumes:
      - /opt/calibre-web/config:/config
      - /srv/share/CalibreLibrary:/books
    ports:
      - "192.168.0.101:8083:8083"
    restart: unless-stopped
```

Start:

```bash
docker compose up -d

# Check logs
docker logs -f calibre-web
```

### Access Calibre-Web

Open `http://192.168.0.101:8083` from your LAN.

First-run setup:
- Set library path to: `/books`
- Create admin user

### Sync from desktop Calibre

On your desktop (after closing Calibre), run:

```bash
rsync -avz --delete "/path/to/DesktopCalibreLibrary/" user@yourserver:/srv/share/CalibreLibrary/
```

Important:
- `--delete` removes books deleted locally (omit if you want to keep server copies)
- Run only when Calibre desktop is closed to avoid database corruption

After sync, refresh the database in Calibre-Web (Settings → Re-scan Library).

### Optional sync script

Create `sync-calibre.sh` on your desktop:

```bash
#!/usr/bin/env bash
set -e
SRC="/path/to/DesktopCalibreLibrary/"
DEST="user@yourserver:/srv/share/CalibreLibrary/"
rsync -avz --delete "$SRC" "$DEST"
echo "Sync complete. Refresh Calibre-Web database."
```

Make executable:

```bash
chmod +x sync-calibre.sh
```

Run manually after editing books in desktop Calibre.

### LAN firewall rule

```bash
sudo iptables -A INPUT -i enp2s0 -p tcp --dport 8083 -j ACCEPT
sudo netfilter-persistent save
```

### Optional reverse proxy via NPM

For external access:

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `books.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `8083`
   - Enable "Websockets Support"
4. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Accept Let's Encrypt Terms
5. Save

### Important notes

- **Only edit with desktop Calibre**. Do not run Calibre GUI container simultaneously.
- **Avoid syncing while Calibre is open** (metadata.db may be mid-write).
- Calibre-Web is read-only by design—use it for browsing/downloading, not editing.

### Summary workflow

1. Edit/manage books on desktop Calibre
2. Close Calibre
3. Run rsync push
4. Refresh Calibre-Web database
5. Browse/download via web interface

## Copyparty - Easy File Browsing & Sharing

Lightweight file server for browsing and uploading files to your shared drives via web interface.

### Create Copyparty directory

```bash
sudo mkdir -p /opt/copyparty
sudo chown -R $USER:$USER /opt/copyparty
cd /opt/copyparty
```

### Create Copyparty docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  copyparty:
    image: copyparty/ac:latest
    container_name: copyparty
    user: "1000:1000"
    ports:
      - "192.168.0.101:3923:3923"
    volumes:
      - /srv/share:/mnt/share
      - /opt/copyparty/config:/cfg
    environment:
      LD_PRELOAD: /usr/lib/libmimalloc-secure.so.NOPE
      # enable mimalloc by replacing "NOPE" with "2" for a nice speed-boost (will use twice as much ram)

      PYTHONUNBUFFERED: 1
      # ensures log-messages are not delayed (but can reduce speed a tiny bit)

    stop_grace_period: 15s  # thumbnailer is allowed to continue finishing up for 10s after the shutdown signal
    healthcheck:
      # hide it from logs with "/._" so it matches the default --lf-url filter 
      test: ["CMD-SHELL", "wget --spider -q 127.0.0.1:3923/?reset=/._"]
      interval: 1m
      timeout: 2s
      retries: 5
      start_period: 15s

    restart: unless-stopped
```

### Create configuration file

Create the configuration file to define your shares and settings:

```bash
vim /opt/copyparty/config/copyparty.conf
```

```ini
# not actually YAML but lets pretend:
# -*- mode: yaml -*-
# vim: ft=yaml:


[global]
  e2dsa  # enable file indexing and filesystem scanning
  e2ts   # enable multimedia indexing
  ansi   # enable colors in log messages (both in logfiles and stdout)
  name: File Server  # name displayed in browser title/header
  xff-src: lan  # trust X-Forwarded-For headers from private IPs (Docker/LAN)
  rproxy: 1     # 1 hop behind reverse proxy

  # q, lo: /cfg/log/%Y-%m%d.log   # log to file instead of docker

  # p: 3939          # listen on another port
  # ipa: 10.89.      # only allow connections from 10.89.*
  # df: 16           # stop accepting uploads if less than 16 GB free disk space
  # ver              # show copyparty version in the controlpanel
  # grid             # show thumbnails/grid-view by default
  # theme: 2         # monokai
  # name: datasaver  # change the server-name that's displayed in the browser
  # stats, nos-dup   # enable the prometheus endpoint, but disable the dupes counter (too slow)
  # no-robots, force-js  # make it harder for search engines to read your server


  /mnt/share           # share /mnt/share (path mounted from host's /srv/share)
  accs:
    rw: *      # everyone gets read-write access, but
    rwmda: ed  # the user "ed" gets read-write-move-delete-admin

```

### Start Copyparty

```bash
docker compose up -d

# Check logs
docker logs -f copyparty
```

### Access Copyparty

Open `http://192.168.0.101:3923` from your LAN.

You'll see your `/srv/share` drive mounted as `/mnt/share` with full browsing and download capabilities.

### LAN firewall rule

```bash
sudo iptables -A INPUT -i enp2s0 -p tcp --dport 3923 -j ACCEPT
sudo netfilter-persistent save
```

### Optional: Expose via NPM

For external access:

1. Open NPM at `http://192.168.0.101:81`
2. Go to "Proxy Hosts" → "Add Proxy Host"
3. **Details tab:**
   - Domain Names: `files.yourdomain.com`
   - Scheme: `http`
   - Forward Hostname/IP: `192.168.0.101`
   - Forward Port: `3923`
   - Enable "Websockets Support"
4. **SSL tab:**
   - SSL Certificate: "Request a new SSL Certificate"
   - Enable "Force SSL"
   - Accept Let's Encrypt Terms
5. Save

Now you can browse files at `https://files.yourdomain.com` from anywhere.

## Watchtower - Container Update Monitoring

Watchtower monitors your Docker containers for updates and can optionally apply them automatically. We'll use **monitor-only mode** to get notifications without automatic updates, giving you control over when to update critical services.

### Create Watchtower directory

```bash
sudo mkdir -p /opt/watchtower
sudo chown -R $USER:$USER /opt/watchtower
cd /opt/watchtower
```

### Create Watchtower docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    environment:
      - TZ=America/Chicago
      - DOCKER_API_VERSION=1.44
      - WATCHTOWER_MONITOR_ONLY=true
      - WATCHTOWER_NOTIFICATION_URL=ntfy://192.168.0.101:8080/watchtower?scheme=http&title=Watchtower
      - WATCHTOWER_NOTIFICATIONS=shoutrrr
      - WATCHTOWER_NOTIFICATIONS_LEVEL=warn
      - WATCHTOWER_SCHEDULE=0 0 17 * * *  # Check daily at 5 PM
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_DISABLE_CONTAINERS=immich_postgres,immich_machine_learning
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

**Configuration explained:**
- `WATCHTOWER_MONITOR_ONLY=true`: Only notify, don't update
- `WATCHTOWER_NOTIFICATIONS`: Send notifications via ntfy
- `WATCHTOWER_NOTIFICATIONS_LEVEL=warn`: Only notify when updates are available (not on every scan)
- `WATCHTOWER_DISABLE_CONTAINERS`: Exclude dependency containers (Immich's postgres and machine-learning) from monitoring
- `WATCHTOWER_SCHEDULE`: Cron schedule (5 PM daily)
- `WATCHTOWER_CLEANUP=true`: Remove old images after updates (if you enable auto-update later)

### Start Watchtower

```bash
docker compose up -d

# Check logs
docker logs -f watchtower
```

### Subscribe to notifications

In ntfy (web or mobile app), subscribe to the `watchtower` topic to receive update notifications.

### Optional: Enable auto-update for specific containers

If you want to auto-update only certain containers, modify the Watchtower config:

```yaml
environment:
  - WATCHTOWER_LABEL_ENABLE=true  # Only update labeled containers
```

Then add labels to containers you trust to auto-update (in their docker-compose.yml):

```yaml
services:
  ntfy:
    image: binwiederhier/ntfy:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

**Recommended for auto-update:**
- ntfy
- Wallabag
- Calibre-Web

**NOT recommended for auto-update:**
- Nextcloud (database migrations need testing)
- Plex (library compatibility)
- Immich (breaking changes common)
- NPM (reverse proxy config changes)

### Manual update workflow

When you receive a notification:

1. Check the container's changelog/release notes
2. Test the update on LAN first
3. Update manually:
   ```bash
   cd /opt/service-name
   docker compose pull
   docker compose up -d
   ```
4. Verify the service still works
5. Clean up old images:
   ```bash
   docker image prune -a
   ```

## Final Firewall Configuration

Since the current firewall rules are messy and contain duplicates, use this script to safely reset them. This method sets the default policy to ACCEPT first to ensure you don't get locked out of SSH when flushing rules.

**Important**: This script flushes ALL rules. It includes lines to restore access for SSH, Samba, Plex, and the Web UIs mentioned in this guide. It also restarts Docker to ensure Docker-managed rules (like WireGuard and NPM public ports) are correctly recreated.

```bash
# 1. Set default policy to ACCEPT to prevent lockout during flush
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT

# 2. Flush all existing rules and delete custom chains
sudo iptables -F
sudo iptables -X

# 3. Add base rules
# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT
# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 4. Add LAN Services (enp2s0)
# Samba (File Sharing)
sudo iptables -A INPUT -i enp2s0 -p udp -m multiport --dports 137,138 -j ACCEPT
sudo iptables -A INPUT -i enp2s0 -p tcp -m multiport --dports 139,445 -j ACCEPT
# Plex (Media Server)
sudo iptables -A INPUT -i enp2s0 -p tcp --dport 32400 -j ACCEPT
# RustDesk (Remote Desktop)
sudo iptables -A INPUT -i enp2s0 -p tcp -m multiport --dports 21115,21116,21117,21118,21119 -j ACCEPT
sudo iptables -A INPUT -i enp2s0 -p udp --dport 21116 -j ACCEPT
# Web UIs (NPM:81, ntfy:8080, Nextcloud:8081, Wallabag:8082, WG:51821, Immich:2283, Calibre-Web:8083, Copyparty:3923)
sudo iptables -A INPUT -i enp2s0 -p tcp -m multiport --dports 81,8080,8081,8082,51821,2283,8083,3923 -j ACCEPT

# 5. Add External Services (eno1)
# Plex (Remote Access)
sudo iptables -A INPUT -i eno1 -p tcp --dport 32400 -j ACCEPT
# RustDesk (Remote Access)
sudo iptables -A INPUT -i eno1 -p tcp -m multiport --dports 21115,21116,21117,21118,21119 -j ACCEPT
sudo iptables -A INPUT -i eno1 -p udp --dport 21116 -j ACCEPT

# 6. Restart Docker to recreate Docker chains and rules
# This handles WireGuard (51820) and NPM (80/443) automatically
sudo systemctl restart docker

# 7. Set default policy back to DROP
sudo iptables -P INPUT DROP

# 8. Save rules
sudo netfilter-persistent save
```

**Note**: Since your server has a direct internet connection via `eno1`, no router port forwarding is needed. The firewall rules above allow direct access from the internet.

## Verifying System Health

If the system freezes or reboots, use these commands to verify everything is back up and running correctly.

### Check Docker Containers

Ensure all containers are running and healthy:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Monitor Resource Usage

Check if any container (specifically `immich_machine_learning`) is consuming excessive resources:

```bash
docker stats
```

If `immich_machine_learning` is using 100% of multiple cores for extended periods, consider applying the resource limits mentioned in the Immich section.

### Check System Logs

Look for any system-level errors that might have occurred during the freeze:

```bash
sudo journalctl -p 3 -xb
```

### Verify Network & Firewall

Ensure network interfaces are up and firewall rules are loaded:

```bash
networkctl status
sudo iptables -L -v -n | head
```

