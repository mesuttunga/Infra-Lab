# Automated Ubuntu Installation (Autoinstall)

Cloud-init based automated installation for cluster nodes.

## Prerequisites

1. **USB Drive**: 8GB+ USB drive
2. **Ubuntu Server ISO**: ubuntu-24.04-live-server-amd64.iso
3. **Balena Etcher**: To write ISO to USB

## Preparation Steps

### 1. Get MAC Addresses

Before creating autoinstall files, get MAC addresses of all nodes:
```bash
# Boot each machine with live USB, open terminal:
ip link show

# Note the MAC address of WiFi interface (wlan0, wlp2s0, etc.)
```

### 2. Generate Password Hash
```bash
# On your local machine:
openssl passwd -6

# Enter password when prompted
# Copy the hash to autoinstall files
```

### 3. Add Your SSH Public Key
```bash
# On your local machine:
cat ~/.ssh/id_ed25519.pub

# Copy the output
# Paste into authorized-keys section in autoinstall files
```

### 4. Update WiFi Credentials

Edit each `tunga-*.yml` file:
- Replace `TUNGA-WIFI` with your WiFi SSID
- Replace `YourWiFiPassword` with your WiFi password

### 5. Update MAC Addresses

Edit each `tunga-*.yml` file:
- Replace `XX:XX:XX:XX:XX:XX` with actual MAC address

## Creating Bootable USB

### Method 1: Using Balena Etcher (Recommended)

1. Download Ubuntu Server ISO
2. Write ISO to USB with Balena Etcher
3. USB will have two partitions after writing

### Method 2: Manual ISO Modification
```bash
# Extract ISO
xorriso -osirrox on -indev ubuntu-24.04-live-server-amd64.iso -extract / iso-extract

# Add autoinstall files
mkdir -p iso-extract/autoinstall
cp tunga-*.yml iso-extract/autoinstall/

# Create new ISO
xorriso -as mkisofs -r \
  -V "Ubuntu 24.04 Autoinstall" \
  -o ubuntu-autoinstall.iso \
  -J -joliet-long \
  -cache-inodes \
  -b isolinux/isolinux.bin \
  -c isolinux/boot.cat \
  -no-emul-boot \
  -boot-load-size 4 \
  -boot-info-table \
  -eltorito-alt-boot \
  -e boot/grub/efi.img \
  -no-emul-boot \
  iso-extract
```

## Installation Process

### Option A: Manual File Selection (Easier)

1. Boot from USB
2. Select "Try Ubuntu" or press `e` on GRUB menu
3. Add to kernel parameters:
```
   autoinstall ds=nocloud-net;s=http://192.168.0.99:8000/
```
4. On your local machine, serve files:
```bash
   cd autoinstall/
   python3 -m http.server 8000
```
5. Machine will download `user-data` (rename appropriate file to user-data)

### Option B: USB-Based (Automated)

1. Copy appropriate autoinstall file to USB root as `user-data`
2. Create empty `meta-data` file on USB root
3. Boot from USB
4. Installation starts automatically

## Post-Installation

After installation completes:

1. **Remove USB and Reboot**
2. **Test SSH Access**:
```bash
   ssh ubuntu@192.168.0.100  # master
   ssh ubuntu@192.168.0.101  # worker1
   ssh ubuntu@192.168.0.102  # worker2
   ssh ubuntu@192.168.0.105  # bastion
```

3. **Run Ansible**:
```bash
   cd ansible/
   ansible-playbook -i inventory.ini setup_cluster.yml
```

## Troubleshooting

### Installation Hangs

- Check WiFi credentials
- Verify MAC address matches
- Ensure network connectivity

### SSH Connection Refused

- Wait 2-3 minutes after first boot
- Verify IP address: `ip addr show`
- Check SSH service: `systemctl status sshd`

### Disk Space Issues

- LVM extension happens in late-commands
- Verify with: `df -h`
- Manual fix: `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`

## Files in This Directory
```
autoinstall/
├── README.md              # This file
├── base.yml              # Template (for reference)
├── tunga-master.yml      # Master node config
├── tunga-worker1.yml     # Worker1 config
├── tunga-worker2.yml     # Worker2 config
└── tunga-bastion.yml     # Bastion config
```

## Security Notes

- Change default password hash in production
- Add all necessary SSH keys before installation
- Passwordless sudo is configured for automation
- Firewall (UFW) will be configured by Ansible