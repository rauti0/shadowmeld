# System Hardening — ShadowMeld (NSM Sensor)

The sensor is hardened at host level with a default-drop firewall, IPv6 disabled, and network-stack sysctl hardening. The goal is a passive monitoring endpoint with minimal attack surface.

Configuration files go under [`config/`](../config) and are deployed to:

- `config/nftables.conf` → `/etc/nftables.conf`
- `config/99-hardening.conf` → `/etc/sysctl.d/99-hardening.conf`
- `config/grub` → `/etc/default/grub`

## 1. Deploy, apply and verify

Put the three files in place and activate them in order. The GRUB change needs a reboot. Run the checks after the host is back up.

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

Verify after reboot:

```bash
# Firewall loaded
sudo nft list ruleset

# IPv6 disabled (kernel-level)
cat /proc/cmdline                                    # contains ipv6.disable=1
ip -6 addr                                           # no IPv6 addresses
cat /proc/sys/net/ipv6/conf/all/disable_ipv6         # No such file or directory

# sysctl hardening active
sudo sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.all.accept_redirects \
            net.ipv4.conf.all.send_redirects net.ipv4.ip_forward

# IPv4 still works
ping -c2 1.1.1.1
```

Expected results: `inet filter` table is loaded; `ipv6.disable=1` in cmdline, no
IPv6 addresses; `/proc/sys/net/ipv6` path does not exist; `rp_filter = 1`,
`accept_redirects = 0`, `send_redirects = 0`, `ip_forward = 0`; ping works.

`nftables.service` loads and exits, so `active (exited)` is the normal healthy state. When IPv6 disabled at the kernel level, `sysctl --system` skips the IPv6 keys with a `-` prefix silently.

## 2. How it works

nftables filters in `table → chain → rule`. A base chain hooks the packet flow
by direction: input (to the host), forward (through it), output (from it). Rules
run top to bottom: the first accept/drop wins, else the chain policy decides. Interface names and 
SSH source are in `define` variables (`$capture_if` = enp4s0, `$mgmt_if` =
enp1s0, `$mgmt_host` = 10.10.10.1).

### Firewall — input chain

    type filter hook input priority filter; policy drop;

Default-drop: anything not matched below is denied.

    iifname $capture_if drop

Drop mirrored traffic first, before conntrack. This keeps the connection table clean. Capture tools 
read via AF_PACKET below netfilter, so this does not affect what is captured.

    iifname "lo" accept
    iifname != "lo" ip daddr 127.0.0.0/8 drop

Allow Loopback traffic. Drop spoofed packets claiming a 127.0.0.0/8 address on any other interface.

    ct state established,related accept
    ct state invalid drop

Allow replies to our own outbound connections (apt, DNS, NTP, DHCP). Drop packets conntrack cannot
recognize.

    iifname $mgmt_if ip saddr $mgmt_host tcp dport 22 ct state new accept

Only inbound service: new SSH connection on `$mgmt_if` from `$mgmt_host`
only. Until the static link exists no host has that address, so SSH stays closed.

### Firewall — forward and output

    type filter hook forward priority filter; policy drop;

The sensor is an endpoint, not a router. Forward chain is empty and drops everything.

    type filter hook output priority filter; policy accept;
    oifname $capture_if drop

Outbound traffic is open (controlled at the workstation), but the host never send on the capture 
interface. This keeps the tap silent.

### IPv6 disabled

    ipv6.disable=1        # /etc/default/grub, GRUB_CMDLINE_LINUX_DEFAULT

Stops the IPv6 stack from starting at all. After reboot `/proc/sys/net/ipv6`
does not exist.

    -net.ipv6.conf.all.disable_ipv6 = 1
    -net.ipv6.conf.default.disable_ipv6 = 1

Backup: the `-` prefix makes sysctl skip these keys silently whhen the cmdline has already disabled
IPv6. If `ipv6.disable=1` is ever removed, these settings disable IPv6 at boot instead.

### Network sysctl hardening

Each key also has a `conf.default` form for interfaces brought up later.

    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.all.accept_source_route = 0

Anti-spoofing: strict reverse-path filtering drops packets whose source
would not route back out the same interface. Source-routed packets are
rejected.

    net.ipv4.conf.all.accept_redirects = 0
    net.ipv4.conf.all.secure_redirects = 0
    net.ipv4.conf.all.send_redirects = 0

ICMP redirects are not accepted (a routing MITM attac vector) and not sent (the host
is not a router).

    net.ipv4.ip_forward = 0

No IP forwarding at stack level. Same as the firewall's forward drop.

    net.ipv4.icmp_echo_ignore_broadcasts = 1
    net.ipv4.icmp_ignore_bogus_error_responses = 1
    net.ipv4.tcp_syncookies = 1

Ignore broadcast pings, drop bad ICMP errors, and
enable SYN cookies against SYN floods.
