# ubuntu-site-to-site-vpn
Create a site-to-site VPN tunnel between an Ubuntu VM and an Azure VPN Gateway.

## Azure VPN Gateway setup

ToDo: provide link(s) to docs describing how to create an active-active Azure VPN Gateway.

**Resultant config**

``` c
// Gateway
psk: "SECRET"

// Instance 0
public ip address: 20.167.80.208
bgp ip address: 10.10.0.12
bgp asn: 65515

// Instance 1
public ip address: 20.167.80.210
bgp ip address: 10.10.0.13
bgp asn: 65515
```

## Azure VM setup

ToDo: provide link to create and Azure VM.

> N.B. Make sure to enable IP forwarding on the VM's NIC resource in Azure.

> N.B. Make sure to allow the required ports through the any NSGs for the VM (500, 4500)

**Resultant config**

``` c
// Vnet
vnet address space: 192.168.0.0/16
subnet address space: 192.168.1.0/29

// Vm
public ip address: 20.28.68.93
ip address: 192.168.1.4
bgp asn: 64512
```

## Ubuntu OS setup

### Pre-requistes

``` bash
# Install packages
sudo apt-get update
sudo apt-get install net-tools

# Configure os settings
sudo sysctl net.ipv4.ip_forward=1
sudo sysctl net.ipv6.conf.all.forwarding=1
sudo sysctl net.ipv4.conf.all.accept_redirects=0
sudo sysctl -p
```

### Tunnel

``` bash
# Install packages
sudo apt-get install strongswan

# Enable services
sudo systemctl enable strongswan-starter

# Backup files
sudo cp /etc/ipsec.secrets /etc/ipsec.secrets.bak
sudo cp /etc/ipsec.conf /etc/ipsec.conf.bak

# Create config
cat <<EOF | sudo tee /etc/ipsec.secrets
20.28.68.93 20.167.80.208 : PSK "SECRET"
EOF

cat <<EOF | sudo tee /etc/ipsec.conf
config setup
    charondebug="all"
    uniqueids=yes
conn azure
    type=tunnel
    auto=start
    keyexchange=ikev2
    authby=secret
    left=192.168.1.4
    leftsubnet=192.168.0.0/16
    leftsourceip=192.168.255.255
    leftid=20.28.68.93
    right=20.167.80.208
    rightsubnet=10.0.0.0/8
    ike=aes256-sha1-modp1024!
    esp=aes256-sha1!
    aggressive=no
    keyingtries=%forever
    ikelifetime=86400s
    lifetime=43200s
    lifebytes=576000000
    dpddelay=30s
    dpdtimeout=120s
    dpdaction=restart
EOF

# Restart service
sudo ipsec restart

# Check connection status
sudo ipsec statusall
```

### BGP

``` bash
# Install packages
sudo apt-get install bird-bgp

# Start services
sudo systemctl enable bird
sudo systemctl restart bird

# Backup files
sudo cp /etc/bird.conf /etc/bird/bird.conf

# Create config
cat <<EOF | sudo tee /etc/bird/bird.conf
log syslog all;
router id 192.168.1.4;
filter unwanted {
    if (net = 0.0.0.0/0) then {
        reject;
    }
    if (net = 168.63.129.16/32) then {
        reject;
    }
    if (net = 169.254.169.254/32) then {
        reject;
    }
    accept;
}
protocol kernel {
    learn;           # learn from the host's route table
    scan time 20;    # Scan kernel routing table every 20 seconds
    export all;      # Default is export none
    import filter unwanted;
}
protocol device {
    scan time 10;    # Scan interfaces every 10 seconds
}
protocol static {
    route 10.10.0.12/32 via "eth0";
}
protocol bgp azure {
    description "BGP Azure VPN Gateway";
    local 192.168.1.4 as 64512;
    neighbor 10.10.0.12 as 65515;
    multihop;
    import all;
    export all;
}
EOF

# Restart serivce
sudo systemctl restart bird

# Check connection status
sudo birdc show status
    sudo birdc show route
```

### Add an example route for testing

``` bash
# Add a route to the OS and see if Azure learns it
sudo ip route add 136.186.0.0/16 dev eth0
```
