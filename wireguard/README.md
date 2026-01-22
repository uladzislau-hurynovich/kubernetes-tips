# WireGuard VPN on AWS EC2
This guide describes how to deploy a WireGuard VPN server on an AWS EC2 instance inside a VPC with split tunneling and support for Route 53 Private Hosted Zones.
---
## Requirements
- AWS EC2 instance (Ubuntu 20.04+)
- EC2 deployed in the target VPC
- Security Group allowing inbound **UDP 51820**
- Source/Destination check disabled for the EC2 instance
---
## Network
- WireGuard CIDR: `10.0.0.0/24`
- AWS VPC DNS resolver: `172.31.0.2`
---

## 1. Install dependencies
```bash
sudo apt update -y
sudo apt install -y wireguard iptables-persistent
```
## 2. Enable IP forwarding
- Enable ip forwarding
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
## 3. Configure NAT
- Detect the primary network interface
```bash
ip address
ip route get 8.8.8.8
```
- Apply NAT (replace ens5 if required)
```bash
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o ens5 -j MASQUERADE
sudo netfilter-persistent save
```
## 4. Generate server keys
```bash
sudo mkdir -p /etc/wireguard
cd /etc/wireguard
wg genkey | sudo tee server_private.key | wg pubkey | sudo tee server_public.key
```
## 5. Configure WireGuard server
- Create **/etc/wireguard/wg0.conf**
```ini
[Interface]
Address = 10.0.0.1/24       # VPN-network
ListenPort = 51820          # WireGuard port
PrivateKey = <server_private.key>
```
## 6. Start WireGuard
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```
Verify
```bash
sudo wg show
```
## 7. Generate client keys
```bash
mkdir -p ~/wireguard-client
cd ~/wireguard-client
wg genkey | tee client_private.key
cat client_private.key | wg pubkey | tee client_public.key
```
## 8. Client configuration
- Create **client.conf**
```ini
[Interface]
PrivateKey = <client_private_key>
Address = 10.0.0.2/24   # IP from VPN-network
DNS = 172.31.0.2          # AmazonProvidedDNS (VPC CIDR + .2)

[Peer]
PublicKey = <server_public_key>
Endpoint = <server_public_ip>:51820
AllowedIPs = 172.16.0.0/12   # Split tunneling (VPC only); use 0.0.0.0/0 for full tunnel
PersistentKeepalive = 25
```
## 9. Add client to server
```bash
sudo wg set wg0 peer <client_public.key> allowed-ips 10.0.0.2/32
```
## 10. Troubleshooting
- WireGuard status
```bash
sudo wg
```
- UDP traffic
```bash
sudo tcpdump -i ens5 udp port 51820
```
- NAT rules
```bash
sudo iptables -t nat -L -n -v
```
## Notes
- `172.31.0.2` is AmazonProvidedDNS and resolves Route 53 Private Hosted Zones
- Only VPC CIDR traffic is routed through the VPN
- Internet traffic is routed directly (split tunneling)
## Security
- Restrict inbound UDP 51820 to trusted IP ranges
- Use unique key pairs per client
- Rotate keys periodically
- Avoid using `SaveConfig = true` in production environments