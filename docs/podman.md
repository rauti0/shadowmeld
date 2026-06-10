# Podman (rootless container runtime)

The monitoring stack (tcpdump, Zeek, Suricata, Prometheus, Grafana) runs in
rootless Podman containers under a dedicated unprivileged service account.
A compromised container gains only that account's privileges, never host root.

## Service account

The stack runs as the unprivileged user `meld` (no sudo, password locked,
no SSH access):

    sudo useradd -m -s /bin/bash meld

Rootless Podman maps container UIDs to a subordinate UID/GID range assigned to
the user in /etc/subuid and /etc/subgid (allocated automatically by useradd):
container UID 0 maps to `meld`, container UID 1+ map into the subordinate range.

Lingering keeps the user's systemd instance running without an active login,
so containers start at boot and survive console/SSH disconnects:

    sudo loginctl enable-linger meld

## Packages

    sudo apt install podman podman-compose uidmap dbus-user-session \
        fuse-overlayfs slirp4netns passt catatonit

- uidmap: newuidmap/newgidmap setuid helpers, required for UID mapping.
- dbus-user-session: user D-Bus, required for systemd --user management.
- fuse-overlayfs: rootless overlay storage driver.
- slirp4netns / passt: rootless networking (passt/pasta is the default).
- catatonit: container init (PID 1).

## Management access

`meld` is reached from `admin`, not by direct login:

    sudo -u meld -i

The shell exports XDG_RUNTIME_DIR and DBUS_SESSION_BUS_ADDRESS (see
~meld/.bashrc) so `podman` and `systemctl --user` work.

## Registries

Trixie ships no unqualified search registries and we keep it that way: always
use fully qualified image names (e.g. docker.io/library/...) for deterministic
pulls. No change to /etc/containers/registries.conf.

## Verify

    sudo -u meld -i
    podman info --format '{{.Host.Security.Rootless}} {{.Store.GraphDriverName}}'   # -> true overlay
    podman run --rm docker.io/library/hello-world
