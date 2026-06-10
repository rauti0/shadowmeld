# Packet Capture with tcpdump — ShadowMeld (NSM Sensor)

Rolling full-packet capture of the mirrored traffic to NVMe. A host systemd
service runs `tcpdump` unprivileged and writes a size-bounded ring of pcap files
to `/data/capture`, which the analytics stack (Zeek, Suricata) reads.

Capture runs on the host, not in a container: opening a raw `AF_PACKET` socket on
the capture interface needs `CAP_NET_RAW` in the initial namespace, which a
rootless container lacks. Rootless Podman runs the analytics instead; capture is
the one small privileged primitive.

Deployable config:

- `config/shadowmeld-capture.service` → `/etc/systemd/system/shadowmeld-capture.service`

The capture interface (`enp4s0`, promiscuous, no IP) is configured in
network-setup.md; the `/data` volume in debian-installation.md.

## Service and privileges

    User=meld
    AmbientCapabilities=CAP_NET_RAW
    CapabilityBoundingSet=CAP_NET_RAW

The service runs as the unprivileged `meld` user, never root — systemd grants
only `CAP_NET_RAW` at exec, enough to open the capture socket and nothing else.
No `setcap` on the binary, so it survives package upgrades. pcaps are owned by
`meld`, so the rootless analytics containers (also `meld`) read them directly
without group or ACL juggling.

The unit is sandboxed (`ProtectSystem=strict` with `ReadWritePaths=/data/capture`,
`NoNewPrivileges`, restricted namespaces and address families). The host never
transmits on the capture interface — egress there is dropped by nftables
(system-hardening.md).

## Capture and rotation

    /usr/bin/tcpdump -p -i enp4s0 -n -s 0 -C 1000 -W 300 -w /data/capture/sensor.pcap

`-p` skips toggling promiscuous mode (the interface is already promiscuous, so
`CAP_NET_ADMIN` is unneeded). `-n` disables name resolution. `-s 0` keeps full
packets for payload analysis.

`-C 1000 -W 300` is a size-based ring: a new file every ~1 GB (`-C` units are
1,000,000 bytes), capped at 300 files (`sensor.pcap000`–`299`). Once full, the
oldest is overwritten — a rolling window of the most recent ~300 GB with a
deterministic disk ceiling. File age comes from each file's mtime (`ls -lt`).
Adjust `-W` for a different budget; leave headroom on `/data` for the OS and
analytics output.

## Deploy and verify

```bash
sudo install -m 644 config/shadowmeld-capture.service /etc/systemd/system/
sudo install -d -o meld -g meld -m 750 /data/capture
sudo systemctl daemon-reload
sudo systemctl enable --now shadowmeld-capture.service
```

```bash
systemctl status shadowmeld-capture.service          # active (running)
ls -lh /data/capture/                                # sensor.pcap000 present, growing
sudo tcpdump -r /data/capture/sensor.pcap000 -c 5 -n # reads back valid packets
```
