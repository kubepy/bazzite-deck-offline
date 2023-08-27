# bazzite-deck-offline

Image List:
https://universal-blue.org/images/

Build the offline image on ArchLinux
```
pacman -Sy tmux cdrtools xorriso cpio skopeo
```

Use `tmux` for building image avoid for disconnecting
```
tmux
```

Download offline bazzite-deck image rename to oci.img (~4.1GB)
```
skopeo copy --all docker://ghcr.io/ublue-os/bazzite-deck:38 oci-archive:oci.img
```


Download origin bazzite iso file (~700MB) and extract files kickstart/pre-install.sh and images/pxeboot/initrd.img
```
curl -LO https://github.com/ublue-os/bazzite/releases/download/v1.0.1/bazzite-38-x86_64-20230822.iso
mount -o loop bazzite-38-x86_64-20230822.iso /mnt
cp -a /mnt/kickstart .
cp -a /mnt/images .
```

Modify initrd.img for removing `checkisomd5@.service` file
```
cd images/pxeboot/
mv initrd.img initrd.img.xz
xz -d initrd.img.xz
cpio -i -d < initrd.img
rm initrd.img vmlinuz etc/systemd/system/checkisomd5\@.service
find . | cpio -H newc --create --verbose | gzip -9 > ~/initrd.img
cd ~
rm -rf images/pxeboot/*
mv initrd.img images/pxeboot/
```

modify pre-install.sh for loading offline oci.img image file insteade of online registry
```
cat << 'EOLEOL' > kickstart/pre-install.sh 
#!/bin/sh

set -oue pipefail

DEFAULT_URL="ghcr.io/ublue-os/silverblue-main:38"

for ARG in `cat /proc/cmdline`; do
    if [[ "${ARG}" =~ ^imageurl= ]]; then
         URL="${ARG#*=}"
    fi
done

URL=$(echo "${URL:-${DEFAULT_URL}}" | tr "[:upper:]" "[:lower:]")

readonly RELEASE="$(sed "2q;d" "/run/install/repo/.discinfo")"
readonly ARCH="$(sed "3q;d" "/run/install/repo/.discinfo")"

cat << EOL > /tmp/ks-urls.txt
ostreecontainer --url="/run/install/repo/oci.img" --transport="oci-archive" --no-signature-verification
url --url="https://download.fedoraproject.org/pub/fedora/linux/releases/${RELEASE}/Everything/${ARCH}/os/"
EOL

# resize /var/tmp for offline image installation
cp -a /var/tmp/* /tmp/
rm -rf /var/tmp
ln -s /tmp /var/tmp
mount -t tmpfs -o remount,size=12G /tmp
EOLEOL
```

Modify bazzite-38-x86_64-*.iso to create bazzite-deck-38_offline_*.iso
```
xorriso -indev bazzite-38-x86_64-20230822.iso -outdev bazzite-deck-38_offline_20230823.iso -boot_image any replay -joliet on -system_id LINUX -compliance joliet_long_names -volid Fedora-E-dvd-x86_64-38 -map oci.img oci.img -map kickstart/pre-install.sh kickstart/pre-install.sh -map images/pxeboot/initrd.img images/pxeboot/initrd.img -end
```

Install ublue-os/bazzite-deck
```
Grub -> Install ublue-os/bazzite-deck
```

(Optional) After install complete and want to change back to online registry
```
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/ublue-os/bazzite-deck:38
```

(Optional) After install complete and want to upgrade from offline image
add `oci-archive` entry to `/etc/containers/policy.json`
```
        "oci-archive": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
```
For example:
```
{
    "default": [
        {
            "type": "reject"
        }
    ],
    "transports": {
        "docker": {
            "registry.access.redhat.com": [
                {
                    "type": "signedBy",
                    "keyType": "GPGKeys",
                    "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"
                }
            ],
            "registry.redhat.io": [
                {
                    "type": "signedBy",
                    "keyType": "GPGKeys",
                    "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"
                }
            ],
            "ghcr.io/ublue-os": [
                {
                    "type": "sigstoreSigned",
                    "keyPath": "/usr/etc/pki/containers/ublue-os.pub",
                    "signedIdentity": {
                        "type": "matchRepository"
                    }
                }
            ],
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        "docker-daemon": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        "atomic": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        "containers-storage": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        "dir": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        "oci": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        "oci-archive": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        "tarball": {
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        }
    }
}
```

Use `sudo rpm-ostree rebase --experimental ostree-unverified-image:oci-archive:/tmp/xxx/oci.img` to load the image
```
sudo rpm-ostree rebase --experimental ostree-unverified-image:oci-archive:/tmp/xxx/oci.img
```
