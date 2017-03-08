# Veer

This script is very hacky way to use VPN for specific domains.
For example, you'd like to listen to Pandora outside of the USA (may or may not
be legal), but you don't want all your connections to go through VPN. Veer will gather
Pandora's IPs and route their traffic through a VPN gateway.

## Requirements:

 - working OpenVPN setup with a config file
 - dig

## Usage

### Install the executable

```sh
$ wget https://raw.githubusercontent.com/ervinb/veer/master/veer -O ~/bin/veer && sudo chmod +x ~/bin/veer
```

### Have a working OpenVPN configuration

```
# host/port of vpn server
remote gw2.iad1.slickvpn.com 443 udp

# verify the server certificate for authenticity
remote-cert-tls server

cipher AES-256-CBC

# equivalent to pull, tls-client
client

# load credentials from a file
## username/email on the first line
## password on the second
auth-user-pass /full/path/to/vpn_creds

proto udp
dev tun
nobind

persist-key
persist-tun

<ca>
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
</ca>
```

### Run

```
veer /path/to/ovpn.conf "www.domain.com"
```

After this, your connection to `www.domain.com` should be routed through VPN.

```sh
$ route

  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  default         csp3.zte.com.cn 0.0.0.0         UG    100    0        0 enp3s0
  10.10.8.1       10.10.8.29      255.255.255.255 UGH   0      0        0 tun0
  10.10.8.29      0.0.0.0         255.255.255.255 UH    0      0        0 tun0
  192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 enp3s0
  192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
  www.domain.com  10.10.8.29      255.255.255.255 UGH   0      0        0 tun0
  www.domain.com  10.10.8.29      255.255.255.255 UGH   0      0        0 tun0
```

### Stop

```
$ pgrep -f '[o]penvpn' | xargs -I PID sudo kill PID
```

## Caveats

### Keeping the connection alive

OpenVPN uses a `tun/tap` device to initate the connection to the VPN server.
There's a mechanism in place in form of `keepalive`, which pings
the VPN server in specific preiods of time, to keep the connection alive. It has a
hidden quirk tough: `keepalive` is a combination of `ping` and `ping-restart`,
which **recreates** the `tun` device, each time ther's no traffic (ping doesn't count as traffic).
Having the `preserve-tun` option isn't helping either.

It is critical to avoid the removal of this device, because by Linux standards,
as soon as a network device dissapears, its routes are gone as well. This is why
`veer` enforces `--ping 15 --ping-restart 0 --preserve-tun` and no `keepalive`.
By doing so, it pings the endpoint, but turns off the `ping-restart` option.

