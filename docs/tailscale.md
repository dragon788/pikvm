---
title: Tailscale VPN
description: How to configure the access to your PiKVM using Tailscale VPN
---

The [Tailscale VPN](https://tailscale.com/) can be used to access PiKVM
from the Internet if configuring [port forwarding](port_forwarding.md)
is not possible or more security is desired. Tailscale is a convenient
and free (for private use) tool for organizing a small VPN network.

The basic Tailscale configuration commands are shown below. For detailed
instructions, refer to [Tailscale
support](https://tailscale.com/contact/support/).

-----

## Configuring the PiKVM

1. Update OS:

    {!_update_os.md!}

2. Install the Tailscale client, run `tailscaled` service and register it in the network:

    ```console
    [root@pikvm ~]# rw
    # If you were afraid to pikvm-update above first run pacman -Syy
    [root@pikvm ~]# pacman -S tailscale-pikvm
    [root@pikvm ~]# systemctl enable --now tailscaled
    [root@pikvm ~]# tailscale up
    ```

3. Follow the link to authorize this installation.
    You likely want to [disable key expiry](https://tailscale.com/kb/1028/key-expiry/)!

4. After authorization success, reboot to make sure that everything works correctly:

    ```console
    [root@pikvm ~]# reboot
    ```

5. Now, you can view the IP address of the Tailscale network interface:

    ```console
    [root@pikvm ~]# ip addr show tailscale0
    ```

If everything is successful, PiKVM will become a member of your VPN network.

!!! warning "Do not update Tailscale if you don't have access to PiKVM without VPN"
    Unfortunately, sometimes, updating the Tailscale client can cause problems due to
    breaking changes. These are compatibility issues on the Tailscale side.
    Remember this when updating.

-----

## Configuring a client device

* [Download](https://tailscale.com/download) and install the Tailscale client
    to the system you are using (not to the system you want to control).
* Check the [Tailscale admin page](https://login.tailscale.com/admin/machines) to view your VPN network.
* Follow the URL in the web browser: `https://<tailscale_kvm_ip>` and you will see the PiKVM web interface.

-----

## Using Tailscale Certificates

PiKVM uses self-signed SSL certificates out of the box. You can also use
[Tailscale certificates](https://tailscale.com/kb/1153/enabling-https) in place of the default one.

!!! warning
    Tailscale certificates are provided by Let's Encrypt and have a default
    [expiry of 90 days](https://letsencrypt.org/2015/11/09/why-90-days/).
    For automatic renewal, see [Automated Renewal Using Persistent Storage
    (PST)](#automated-renewal-using-persistent-storage-pst) below.
    The manual steps in this section are useful for a one-time setup or
    if you prefer not to use the automated approach.

1. Switch filesystem to RW if in ReadOnly mode and delete existing PiKVM certificates for nginx and vnc.

    ```console
    [root@pikvm ~]# rw
    [root@pikvm ~]# rm -v /etc/kvmd/{nginx,vnc}/ssl/*.{crt,key}
    ```

2. Provision new certificates using [`tailscale cert`](https://tailscale.com/kb/1080/cli#cert)
    command. Optionally you may create a directory to store the certificates.

    ```console
    [root@pikvm ~]# mkdir .cert
    [root@pikvm ~]# cd .cert
    [root@pikvm .cert]# tailscale cert <tailscale_hostname>
    ```

3. Copy the certificates to nginx's and vnc's ssl directories.

    ```console
    [root@pikvm ~]# cp /root/.cert/<tailscale_hostname>.key /etc/kvmd/nginx/ssl/server.key
    [root@pikvm ~]# cp /root/.cert/<tailscale_hostname>.crt /etc/kvmd/nginx/ssl/server.crt
    ```

    Repeat the same steps for vnc if you have configured it.

    ```console
    [root@pikvm ~]# cp /root/.cert/<tailscale_hostname>.key /etc/kvmd/vnc/ssl/server.key
    [root@pikvm ~]# cp /root/.cert/<tailscale_hostname>.crt /etc/kvmd/vnc/ssl/server.crt
    ```

4. Grant file ownership to nginx and vnc services. Switch filesystem to ReadOnly again

    ```console
    [root@pikvm ~]# chown :kvmd-nginx /etc/kvmd/nginx/ssl/*
    [root@pikvm ~]# chown :kvmd-vnc /etc/kvmd/vnc/ssl/*
    [root@pikvm ~]# ro
    ```

5. Restart nginx and vnc services

    ```console
    [root@pikvm ~]# systemctl restart kvmd-nginx
    [root@pikvm ~]# systemctl restart kvmd-vnc
    ```

-----

## Automated Ephemeral Tailscale Certificates Renewal

Tailscale has a nice option of running an HTTPS on your behalf within your tailnet: [`tailscale serve`](https://tailscale.com/kb/1312/serve). It is using Let's Encrypt certificates and renews them every 90 days. The issue is that PiKVM’s filesystem is read-only. While tailscale will diligently request new certificates, it will fail to write it on the disk and hence will try to request new certificates next time you access your web server. Let's Encrypt has a limit of 5 certificates for the server per week, so you will end up with an inoperable server and rate-limited by Let's Encrypt for a day or so.

Here's the command that allows you to seamlessly run HTTPS proxy for your PiKVM:
```console
[root@pikvm ~]# tailscale serve --bg https+insecure://localhost:443
```
And if you want to stop tailscale from serving HTTPS, you can do this by running:
```console
[root@pikvm ~]# tailscale serve --https=443 off
```

### Root cause
Tailscale needs to refresh TLS certificates and write state under `/var/lib/tailscale`.  
On PiKVM, the root filesystem is read-only, so direct writes fail.  

We can fix this by mounting an **ephemeral overlay filesystem (tmpfs) in RAM** for `/var/lib/tailscale`, backed by a persistent lowerdir (`/root/tailscale-state`).

This ensures that certificate rotation and state writes work without breaking PiKVM’s read-only state.

!!! warning
    The **caveat** is that renewed certificates exist only in RAM. After a reboot, Tailscale falls back to the older certificates on disk, requests fresh ones, and stores them in RAM again.
    If you reboot PiKVM too frequently, this can trigger Let's Encrypt's rate limits.

### Solution

Core idea:
- Mount a **tmpfs** over Tailscale's state folder stored in root's home: /root/tailscale-state. 
- Mount the resulting *merged* layer onto the actual Tailscale state folder at /var/lib/tailscale.
- An **overlayfs** will transparently present this folder to Tailscale, while changes are kept in the RAM-based overlay layer.

**Note**: Overlayfs requires that the upperdir and workdir exist before creating the overlay.
Since these directories live in RAM, they disappear after every reboot.
This means we cannot use fstab to declare the mount points.
Instead, we implement this with a systemd service that runs a setup script during boot, before tailscaled starts.

1. Switch filesystem to RW and copy Tailscale state:

```console
[root@pikvm ~]# rw
[root@pikvm ~]# cp -a /var/lib/tailscale /root/tailscale-state
````

2. Create a helper script, save as `/usr/local/bin/setup-tailscale-overlay.sh`:

```bash
#!/bin/bash
set -e

# Make tmpfs for tailscale overlay
mkdir -p /tmp/tailscale-tmpfs
mountpoint -q /tmp/tailscale-tmpfs || mount -t tmpfs tmpfs /tmp/tailscale-tmpfs

# Prepare overlay dirs
mkdir -p /tmp/tailscale-tmpfs/upper
mkdir -p /tmp/tailscale-tmpfs/work
mkdir -p /tmp/tailscale-merged

# Mount overlay (lowerdir = persistent readonly state in /root)
mountpoint -q /tmp/tailscale-merged || mount -t overlay overlay \
  -o lowerdir=/root/tailscale-state,upperdir=/tmp/tailscale-tmpfs/upper,workdir=/tmp/tailscale-tmpfs/work \
  /tmp/tailscale-merged

# Bind merged to /var/lib/tailscale
mountpoint -q /var/lib/tailscale && umount /var/lib/tailscale || true
mount --bind /tmp/tailscale-merged /var/lib/tailscale
```

Make it executable:

```console
[root@pikvm ~]# chmod +x /usr/local/bin/setup-tailscale-overlay.sh
```

3. Create a systemd unit

We need to run the overlay setup **after `/tmp` is mounted** but **before `tailscaled.service`**.

Save as `/etc/systemd/system/tailscale-overlay.service`:

```ini
[Unit]
Description=Setup overlayfs for Tailscale
After=local-fs.target tmp.mount
Before=tailscaled.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup-tailscale-overlay.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

**Notes:**

* `local-fs.target` ensures all local filesystems (including `/tmp` tmpfs from fstab) are mounted.
* `tmp.mount` is added explicitly in case your system defines it.
* Overlay is mounted and bound before `tailscaled` starts.

4. Enable and reload

```console
[root@pikvm ~]# systemctl daemon-reload
[root@pikvm ~]# systemctl enable tailscale-overlay.service
[root@pikvm ~]# ro
```

---

### Boot sequence recap:

1. tmpfs is mounted at `/tmp/tailscale-tmpfs`
2. `upper` + `work` dirs are recreated inside tmpfs
3. overlay is mounted with `/root/tailscale-state` as lowerdir
4. overlay bind-mounted to `/var/lib/tailscale`
5. `tailscaled.service` starts with writable state

-----

## Automated Renewal Using Persistent Storage (PST)

An alternative to the overlay approach above is to store Tailscale
certificates on PiKVM's [persistent storage (PST)](pst.md) partition
and use a systemd timer for automatic renewal. This avoids
tmpfs/overlayfs entirely.

!!! warning "Use `kvmd-pstrun`, not a manual remount"
    Always write to PST through `kvmd-pstrun -- <command>`, never by
    calling `kvmd-helper-pst-remount rw`/`ro` yourself. The `kvmd-pst`
    daemon reference-counts access and keeps the partition writable only
    while a client needs it — a manual remount races with any other PST
    user (for example the KVM switch settings writer or a certbot job)
    and one caller's `ro` can pull the mount out from under another
    mid-write. `kvmd-pstrun` remounts RW for the duration of your
    command and drops back to RO automatically when it exits.

The `tailscale cert` command is safe to run repeatedly — if the
certificate is still valid, it exits immediately without contacting
Let's Encrypt, so there is no risk of hitting rate limits.

### Setup

1. Create a directory for the certificates on the PST partition:

    ```console
    [root@pikvm ~]# kvmd-pstrun -- mkdir -p /var/lib/kvmd/pst/data/tailscale-certs
    ```

2. Generate the initial certificate. Replace `<tailscale_hostname>`
    with your PiKVM's Tailscale FQDN (e.g.
    `mypikvm.my-tailnet.ts.net`). If you are unsure, run
    `tailscale cert` with no arguments and it will show the correct
    domain.

    ```console
    [root@pikvm ~]# CERT_DIR=/var/lib/kvmd/pst/data/tailscale-certs
    [root@pikvm ~]# kvmd-pstrun -- tailscale cert \
        --cert-file "$CERT_DIR/<tailscale_hostname>.crt" \
        --key-file "$CERT_DIR/<tailscale_hostname>.key" \
        <tailscale_hostname>
    [root@pikvm ~]# kvmd-pstrun -- chown :kvmd-nginx "$CERT_DIR"/<tailscale_hostname>.{crt,key}
    [root@pikvm ~]# kvmd-pstrun -- chmod 644 "$CERT_DIR/<tailscale_hostname>.crt"
    [root@pikvm ~]# kvmd-pstrun -- chmod 640 "$CERT_DIR/<tailscale_hostname>.key"
    ```

3. Replace the default self-signed certificates with symlinks to the
    PST-stored certificates. These symlinks live under `/etc` on the
    **root** filesystem (not PST), so this step uses the normal
    `rw`/`ro` root remount and only needs to be done once:

    ```console
    [root@pikvm ~]# rw
    [root@pikvm ~]# rm -f /etc/kvmd/nginx/ssl/server.{crt,key}
    [root@pikvm ~]# ln -s /var/lib/kvmd/pst/data/tailscale-certs/<tailscale_hostname>.crt /etc/kvmd/nginx/ssl/server.crt
    [root@pikvm ~]# ln -s /var/lib/kvmd/pst/data/tailscale-certs/<tailscale_hostname>.key /etc/kvmd/nginx/ssl/server.key
    [root@pikvm ~]# chown -h :kvmd-nginx /etc/kvmd/nginx/ssl/server.{crt,key}
    ```

    Repeat for VNC if you have configured it:

    ```console
    [root@pikvm ~]# rm -f /etc/kvmd/vnc/ssl/server.{crt,key}
    [root@pikvm ~]# ln -s /var/lib/kvmd/pst/data/tailscale-certs/<tailscale_hostname>.crt /etc/kvmd/vnc/ssl/server.crt
    [root@pikvm ~]# ln -s /var/lib/kvmd/pst/data/tailscale-certs/<tailscale_hostname>.key /etc/kvmd/vnc/ssl/server.key
    [root@pikvm ~]# chown -h :kvmd-vnc /etc/kvmd/vnc/ssl/server.{crt,key}
    ```

    Leave the root filesystem in RW mode for now — the remaining setup
    steps also write to it. You will switch back to RO at the end.

4. Create the renewal script. Save as
    `/usr/local/bin/renew-tailscale-cert.sh`:

    ```bash
    #!/bin/bash
    set -euo pipefail

    # Your PiKVM's Tailscale FQDN, e.g. mypikvm.my-tailnet.ts.net
    # Recommended: set it explicitly. Leave empty to auto-detect.
    FQDN=""

    export CERT_DIR=/var/lib/kvmd/pst/data/tailscale-certs

    if [ -z "$FQDN" ]; then
        # Self.DNSName from Tailscale's own status output, minus the
        # trailing dot. Uses jq when available, otherwise falls back.
        if command -v jq >/dev/null; then
            FQDN=$(tailscale status --json | jq -r '.Self.DNSName')
        else
            FQDN=$(tailscale status --json \
                | grep -m1 -o '"DNSName"[^"]*"[^"]*"' | cut -d'"' -f4)
        fi
        FQDN=${FQDN%.}
    fi

    case "$FQDN" in
        *.ts.net) ;;
        *)
            echo "ERROR: could not determine Tailscale FQDN." >&2
            echo "Set FQDN manually at the top of this script." >&2
            exit 1
            ;;
    esac
    export FQDN

    # Write to PST through the kvmd-pst daemon (race-safe RW remount).
    # tailscale cert resets ownership/mode on overwrite, so re-apply
    # them in the same session while the partition is writable.
    kvmd-pstrun -- bash -c '
        set -e
        tailscale cert \
            --cert-file "$CERT_DIR/$FQDN.crt" \
            --key-file "$CERT_DIR/$FQDN.key" \
            "$FQDN"
        chown :kvmd-nginx "$CERT_DIR/$FQDN.crt" "$CERT_DIR/$FQDN.key"
        chmod 644 "$CERT_DIR/$FQDN.crt"
        chmod 640 "$CERT_DIR/$FQDN.key"
    '

    systemctl restart kvmd-nginx kvmd-vnc
    echo "Cert renewed for $FQDN"
    ```

    Make it executable:

    ```console
    [root@pikvm ~]# chmod +x /usr/local/bin/renew-tailscale-cert.sh
    ```

5. Create the systemd service. Save as
    `/etc/systemd/system/renew-tailscale-cert.service`:

    ```ini
    [Unit]
    Description=Renew Tailscale TLS certificate
    After=network-online.target tailscaled.service
    Requires=tailscaled.service
    Wants=network-online.target

    [Service]
    Type=oneshot
    ExecStart=/usr/local/bin/renew-tailscale-cert.sh

    [Install]
    WantedBy=multi-user.target
    ```

6. Create the systemd timer for weekly renewal. Save as
    `/etc/systemd/system/renew-tailscale-cert.timer`:

    ```ini
    [Unit]
    Description=Tailscale TLS certificate renewal (weekly)

    [Timer]
    OnCalendar=weekly
    RandomizedDelaySec=3600
    Persistent=true

    [Install]
    WantedBy=timers.target
    ```

7. Enable everything, then switch the root filesystem back to
    read-only:

    ```console
    [root@pikvm ~]# systemctl daemon-reload
    [root@pikvm ~]# systemctl enable renew-tailscale-cert.service
    [root@pikvm ~]# systemctl enable renew-tailscale-cert.timer
    [root@pikvm ~]# ro
    ```

The service runs at boot (after `tailscaled` and networking are up)
and the timer triggers a weekly renewal. Since `tailscale cert` is a
no-op when the certificate is still valid, this will only contact
Let's Encrypt when the certificate is approaching expiry.

!!! note
    The `Persistent=true` option on the timer ensures that if PiKVM
    was powered off when a scheduled renewal was due, the renewal
    will run at the next boot.

-----

## NFS Root Filesystem

!!! note
    This section applies only to uncommon custom setups where PiKVM's
    root filesystem is served over NFS. Most users can skip it.

If your PiKVM boots from an NFS root, the read-only constraint that
motivates both the overlayfs and PST approaches above does not apply
— the NFS-mounted root is already writable. However, there are still
a few things to be aware of.

**Tailscale state** (`/var/lib/tailscale`) can be written directly on
an NFS root without any overlay or remount steps. The overlayfs
workaround is not needed.

**Certificate renewal** can similarly write directly to any path on
the NFS share. The `tailscale cert` command and the systemd timer from
the PST section above still apply, but you can skip the `kvmd-pstrun`
wrapper and write certs to any persistent path you prefer (e.g.
`/etc/kvmd/tailscale-certs/`) — point `CERT_DIR` in the renewal script
there and invoke `tailscale cert` directly instead of through
`kvmd-pstrun`.

**Overlayfs on NFS lowerdirs** is not supported on older kernels and
should be avoided regardless. If you are using NFS root and attempted
the overlayfs approach, switch to direct writes instead.

**`rw`/`ro` commands** may behave differently or not apply at all
depending on how your NFS root is configured. Check your specific
setup before following any steps that call `rw` or `ro`.

-----

## Troubleshooting

* If something does not work, the usual advice is to completely remove Tailscale from PiKVM and perform a clean installation:

    ```console
    [root@pikvm ~]# rw
    [root@pikvm ~]# pacman -Rscnd tailscale
    [root@pikvm ~]# rm -rf /var/lib/tailscale /var/cache/tailscale
    [root@pikvm ~]# reboot
    ```

Now, follow the instructions from the beginning to re-install Tailscale.

* In case of certificate issues you can try the following steps to debug and fix.

1. Check if the services are running. If not please start them. For example,
    web UI service can be checked using:

    ```console
    [root@pikvm ~]# systemctl status kvmd-nginx
    ```

    For VNC:

    ```console
    [root@pikvm ~]# systemctl status kvmd-vnc
    ```

2. If the services are running but not accessible or showing a warning, check
    the respective logs. For web UI:

    ```console
    [root@pikvm ~]# journalctl -xeu kvmd-nginx
    ```

3. If the logs shows TLS/certificate/permissions errors, the issue may be with
    file ownership. The services must have at least the group ownership of the
    certificates. The ownership should look similar to this:

    ```console
    [root@pikvm ~]# ls -l /etc/kvmd/{nginx,vnc}/ssl
    /etc/kvmd/nginx/ssl:
    total 8
    -r--r--r-- 1 root kvmd-nginx 2872 Jan  3 16:07 server.crt
    -r--r----- 1 root kvmd-nginx  227 Jan  3 16:07 server.key

    /etc/kvmd/vnc/ssl:
    total 8
    -r--r--r-- 1 root kvmd-vnc 2872 Jan  3 16:07 server.crt
    -r--r----- 1 root kvmd-vnc  227 Jan  3 16:07 server.key
    ```
