# Zeek

Network security monitor running as a rootful Podman container managed by
Quadlet. Sniffs the capture interface (enp4s0) independently of tcpdump and
writes logs to /data/zeek.

## Image

docker.io/zeek/zeek:lts — empty entrypoint, default cmd `bash`, so `zeek` is
invoked explicitly as the container command.

## Site configuration

config/zeek/site.zeek, deployed to /data/zeek/site.zeek and loaded by absolute
path:

    @load local
    redef Log::default_rotation_interval = 1 hr;

JSON logging is available but disabled (commented `redef LogAscii::use_json`).

## Quadlet unit

/etc/containers/systemd/zeek.container:

- Network=host (sniffs the host interface directly)
- DropCapability=ALL, then AddCapability NET_RAW + NET_ADMIN only
- Volume /data/zeek (no :Z — Debian uses AppArmor, not SELinux)
- WorkingDir /data/zeek (log output directory)
- Exec: zeek -i enp4s0 /data/zeek/site.zeek
- Restart=on-failure (bring-up; switch to always for 24/7 — TODO)

## Activation

    sudo systemctl daemon-reload
    sudo systemctl start zeek.service

Generated service is zeek.service; `systemctl is-enabled` reports "generated"
and it starts at boot via the [Install] section.

## Verification

Confirmed conn.log is created under /data/zeek and packet stats show 0 dropped.

