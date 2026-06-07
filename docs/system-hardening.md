# System Hardening — ShadowMeld (NSM Sensor)

Host-level hardening for the sensor: a default-drop firewall, IPv6 disabled, and
network-stack sysctl hardening. The goal is a passive monitoring endpoint with a
minimal attack surface — minimal exposure on the network, and it never acts as
a router.

Configuration files live under [`config/`](../config) and deploy to:

- `config/nftables.conf` → `/etc/nftables.conf`
- `config/99-hardening.conf` → `/etc/sysctl.d/99-hardening.conf`
- `config/grub` → `/etc/default/grub`

## 1. Deploy, apply and verify

Put the three files in place, then activate them in order. The GRUB change needs
a reboot, which also completes the kernel-level IPv6 disable — run the checks at
the end once the host is back up.

Firewall — syntax-check, enable at boot, load now:

```bash
sudo nft -c -f /etc/nftables.conf      # no output means OK
sudo systemctl enable nftables
sudo systemctl restart nftables
```

sysctl hardening — apply all `/etc/sysctl.d/*.conf`:

```bash
sudo sysctl --system
```

Kernel IPv6 disable — add `ipv6.disable=1` to `GRUB_CMDLINE_LINUX_DEFAULT` in
`/etc/default/grub`, then regenerate and reboot:

```bash
sudo update-grub
sudo reboot
```

Verify once the host is back up:

```bash
# Firewall loaded
sudo nft list ruleset

# IPv6 stack absent (kernel-level disable)
cat /proc/cmdline                                   # contains ipv6.disable=1
ip -6 addr                                           # no IPv6 addresses
cat /proc/sys/net/ipv6/conf/all/disable_ipv6         # No such file or directory

# sysctl hardening active
sudo sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.all.accept_redirects \
            net.ipv4.conf.all.send_redirects net.ipv4.ip_forward

# IPv4 connectivity intact
ping -c2 1.1.1.1
```

Expected: the `inet filter` table present; `ipv6.disable=1` in the cmdline, no
IPv6 addresses, and the missing `/proc/sys/net/ipv6` path; `rp_filter = 1`,
`accept_redirects = 0`, `send_redirects = 0`, `ip_forward = 0`; ping succeeds.

`nftables.service` is a oneshot — it loads the ruleset and exits, so
`active (exited)` is the normal healthy state. With IPv6 disabled at the kernel
level, `sysctl --system` skips the `-`-prefixed IPv6 keys silently.

## 2. How it works

nftables filters in `table → chain → rule`. A base chain hooks into the packet
path by direction — input (to this host), forward (through it), output (from it).
Within a chain, rules run top to bottom: the first accept/drop wins, otherwise
the chain policy decides. Interface names and the SSH source live in `define`
variables (`$capture_if` = enp4s0, `$mgmt_if` = enp1s0, `$mgmt_host` =
10.10.10.1), matched by name so the ruleset loads even before the links are up.

### Firewall — input chain

    type filter hook input priority filter; policy drop;

Base chain on the input hook; default-drop means anything not matched below is
denied.

    iifname $capture_if drop

Mirrored traffic is dropped first, before conntrack, so the connection table
never fills with flows that aren't ours. Capture tools read via AF_PACKET below
netfilter, so this does not affect what is captured.

    iifname "lo" accept
    iifname != "lo" ip daddr 127.0.0.0/8 drop

Loopback is allowed; a packet on any other interface claiming a 127.0.0.0/8
destination is spoofed and dropped.

    ct state established,related accept
    ct state invalid drop

Replies to our own outbound connections (apt, DNS, NTP, DHCP renewal) and the
ICMP errors tied to them (including PMTUD) return statefully; packets conntrack
cannot place are dropped.

    iifname $mgmt_if ip saddr $mgmt_host tcp dport 22 ct state new accept

The only inbound service: a new SSH connection, on the management interface, from
the workstation's address only. Until the static link exists no host holds that
address, so SSH stays closed and management runs over serial.

### Firewall — forward and output

    type filter hook forward priority filter; policy drop;

The sensor is an endpoint, not a router; the forward chain is empty and drops.

    type filter hook output priority filter; policy accept;
    oifname $capture_if drop

Egress is open (policed at the workstation), but the host never transmits on the
capture interface, keeping the tap silent.

### IPv6 disabled

    ipv6.disable=1        # /etc/default/grub, GRUB_CMDLINE_LINUX_DEFAULT

Keeps the IPv6 stack from initialising at all; after reboot `/proc/sys/net/ipv6`
does not exist.

    -net.ipv6.conf.all.disable_ipv6 = 1
    -net.ipv6.conf.default.disable_ipv6 = 1

The backstop: the leading `-` makes sysctl skip the key silently while the
cmdline disable has removed that path. If `ipv6.disable=1` is ever dropped, the
stack returns and these disable IPv6 at boot instead.

### Network sysctl hardening

Each key also has a `conf.default` form for interfaces brought up later.

    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.all.accept_source_route = 0

Anti-spoofing: strict reverse path filtering drops a packet whose source would
not route back out the interface it arrived on; source-routed packets are
rejected.

    net.ipv4.conf.all.accept_redirects = 0
    net.ipv4.conf.all.secure_redirects = 0
    net.ipv4.conf.all.send_redirects = 0

ICMP redirects are not accepted (a routing-MITM vector) and not sent (the host is
not a router).

    net.ipv4.ip_forward = 0

No forwarding at the stack level — the same stance as the firewall's forward drop.

    net.ipv4.icmp_echo_ignore_broadcasts = 1
    net.ipv4.icmp_ignore_bogus_error_responses = 1
    net.ipv4.tcp_syncookies = 1

Ignore broadcast pings (smurf protection), drop malformed ICMP errors, and enable
SYN cookies against SYN floods.
