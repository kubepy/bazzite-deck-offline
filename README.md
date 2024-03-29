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
skopeo copy --all --remove-signatures docker://ghcr.io/ublue-os/bazzite-deck-gnome:39 oci-archive:oci.img
```


Download origin bazzite iso file (~700MB) and extract files kickstart/pre-install.sh and images/pxeboot/initrd.img
```
curl -LO https://github.com/ublue-os/bazzite/releases/download/v2.2.0/bazzite-39-x86_64-20240119.iso
mount -o loop bazzite-39-x86_64-20240119.iso /mnt
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

DEFAULT_URL="ghcr.io/ublue-os/silverblue-main:39"

for ARG in `cat /proc/cmdline`; do
    if [[ "${ARG}" =~ ^imageurl= ]]; then
         URL="${ARG#*=}"
    fi
done

URL=$(echo "${URL:-${DEFAULT_URL}}" | tr "[:upper:]" "[:lower:]")

RELEASE="$(sed "2q;d" "/run/install/repo/.discinfo")"
[[ "${RELEASE}" -eq "39" ]] && RELEASE="latest"
readonly RELEASE

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

Modify bazzite-39-x86_64-*.iso to create bazzite-deck-gnome-39-x86_64-20240119_offline.iso
```
xorriso -indev bazzite-39-x86_64-20240119.iso -outdev bazzite-deck-gnome-39-x86_64-20240119_offline.iso -boot_image any replay -joliet on -system_id LINUX -compliance joliet_long_names -volid Fedora-E-dvd-x86_64-39 -map oci.img oci.img -map kickstart/pre-install.sh kickstart/pre-install.sh -map images/pxeboot/initrd.img images/pxeboot/initrd.img -end
```

Install ublue-os/bazzite-deck
```
Grub -> Install ublue-os/bazzite-deck-gnome
```

(Optional) After install complete and want to change gamescope mode autologin
```
just enable-gamescope-autologin
```

(Optional) After install complete and want to change back to online registry
```
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/ublue-os/bazzite-deck-gnome:39
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

Use `sudo rpm-ostree rebase --experimental ostree-unverified-image:oci-archive:/tmp/oci.img` to load the image
```
sudo setenforce 0
sudo rpm-ostree rebase --experimental ostree-unverified-image:oci-archive:/tmp/oci.img
```
