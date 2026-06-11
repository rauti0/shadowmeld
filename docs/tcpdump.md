# Packet Capture with tcpdump — ShadowMeld (NSM Sensor)

Rolling full-packet capture of the mirrored traffic to NVMe. A
systemd service runs `tcpdump` as a normal user and writes hourly pcap
files to `/data/pcap`. The analytics stack (Zeek, Suricata) reads them.

Capture runs on the host, not in a container. Opening a raw capture
socket needs `CAP_NET_RAW` on the host, and a rootless container does not
have it. Capture is the one small privileged piece, and everything
else runs rootless.

Deployable config:
- `config/tcpdump/tcpdump-capture`           → `/usr/local/bin/tcpdump-capture`
- `config/tcpdump/tcpdump-capture.service`   → `/etc/systemd/system/tcpdump-capture.service`
- `config/tcpdump/tcpdump-retention`         → `/usr/local/bin/tcpdump-retention`
- `config/tcpdump/tcpdump-retention.service` → `/etc/systemd/system/tcpdump-retention.service`
- `config/tcpdump/tcpdump-retention.timer`   → `/etc/systemd/system/tcpdump-retention.timer`

The capture interface (enp4s0, promiscuous, no IP) is set up in
`network-setup.md` and the `/data` volume in `debian-installation.md`.

## Service and privileges

    User=meld
    AmbientCapabilities=CAP_NET_RAW
    CapabilityBoundingSet=CAP_NET_RAW

The service runs as the meld user, never root. `systemd` gives the
process only `CAP_NET_RAW`, to open the capture socket and nothing else. 
No setcap on the binary, so package upgrades cannot drop the capability.
The pcap files are owned by meld, and the rootless analytics containers run as meld too, 
so they can read the files directly.

The unit is sandboxed: `ProtectSystem=strict` with
`ReadWritePaths=/data/pcap`, NoNewPrivileges, restricted namespaces and
address families. The host never sends anything on the capture
interface — `nftables` drops all egress there (`system-hardening.md`).

## Time

The host runs in fixed UTC+3 (Etc/GMT-3, no DST). 
File names, mtimes and journald all match. The Zeek container mounts `/etc/localtime`
read-only so its log names match too.

Rotation may lag a few seconds behind the hour. tcpdump rotates on the
first packet after the boundary, and the offset persists in later file
names. Packet timestamps inside the files are always exact.

## Capture and rotation

The service runs a wrapper (`/usr/local/bin/tcpdump-capture`, copy in
config/tcpdump/) around a single tcpdump command:

    tcpdump -p -i enp4s0 -n -s 0 -G 3600 -C 1000 -w /data/pcap/sensor_%Y%m%d_%H%M%S.pcap

The wrapper starts tcpdump twice. The first run stops at the next full hour. 
The second run starts right at the full hour, so every file starts at HH:00 and covers one hour:
`sensor_YYYYMMDD_HH0000.pcap.`

- `-p`       don't toggle promiscuous mode; the interface is already promiscuous
- `-n`       no name resolution
- `-s 0`     full packets
- `-G 3600`  rotate hourly
- `-C 1000`  spike safety net: split at ~1 GB for retention; never triggers at ~100 MB/h
- `-w`       filename from capture start time

## Retention

Script: `/usr/local/bin/tcpdump-retention`.
Deletes the oldest capture files until the total size of
/data/pcap/sensor_*.pcap* is under the budget (default 300 GB).
The newest file is never deleted because tcpdump is writing to it.

Runs every 15 minutes via systemd timer:
- `tcpdump-retention.service` (oneshot, User=meld, sandboxed)
- `tcpdump-retention.timer` (OnCalendar=*:0/15)

Override the budget for testing: `BUDGET_GB=<n> tcpdump-retention`.
Check runs: `sudo journalctl -u tcpdump-retention.service`

## Deploy

    sudo install -m 755 config/tcpdump/tcpdump-capture /usr/local/bin/
    sudo install -m 644 config/tcpdump/tcpdump-capture.service /etc/systemd/system/
    sudo install -m 755 config/tcpdump/tcpdump-retention /usr/local/bin/
    sudo install -m 644 config/tcpdump/tcpdump-retention.service /etc/systemd/system/
    sudo install -m 644 config/tcpdump/tcpdump-retention.timer /etc/systemd/system/
    
    sudo systemctl enable --now tcpdump-capture.service
    sudo systemctl enable --now tcpdump-retention.service
    sudo systemctl enable --now tcpdump-retention.timer
   
    sudo install -d -o meld -g meld -m 750 /data/pcap
    sudo systemctl daemon-reload

## Verify

    systemctl status tcpdump-capture.service                    # active (running)
    ls -lh /data/pcap/                                          # sensor_*.pcap present, growing
    sudo tcpdump -r /data/pcap/sensor_<latest>.pcap -c 5 -n     # reads back valid packets
