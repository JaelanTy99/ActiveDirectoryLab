# Phase 1 — VM Install & Host Setup

**Host:** Fedora Linux 44 desktop
**Hypervisor:** KVM/QEMU via `virt-install` + `virt-manager` (native to Fedora — chosen over VirtualBox to avoid RPM Fusion / kernel-module-signing friction on every kernel update)
**Guest:** Windows Server 2022 Standard Evaluation (Desktop Experience), 180-day eval, no product key required

## 1. Host Pre-Flight

Verified hardware virtualization was enabled and the host had headroom before installing anything:

```bash
grep -Ec '(vmx|svm)' /proc/cpuinfo   # confirmed 32 (cores with the flag)
free -h
nproc
df -h /var/lib/libvirt/images
```

## 2. Hypervisor Install

```bash
sudo dnf install -y qemu-kvm libvirt virt-install virt-manager virt-viewer \
  libvirt-daemon-config-network libvirt-daemon-kvm edk2-ovmf

sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt "$USER"
```

**Issue encountered:** group membership change didn't take effect in the existing terminal session even after a logout that turned out to be a lock-screen, not a true session logout. Fixed with `newgrp libvirt` to activate the new group in-session without needing a full reboot.

## 3. Media Acquisition

- **Windows Server 2022 Evaluation ISO** — Microsoft Evaluation Center (`~4.7 GB`)
- **VirtIO drivers** — required because the Windows installer has no built-in driver for `virtio` disks/NICs

**Issue encountered:** the commonly documented direct-download URL for virtio-win (`fedorapeople.org/.../stable-virtio/virtio-win.iso`) returned a 404 — link rot on the upstream hosting path. Root-caused by fetching the file directly and confirming the response body was a genuine "file not found" page rather than a network/proxy block. Resolved by using Fedora's own package repo instead of chasing a raw URL:

```bash
wget -qO- https://fedorapeople.org/groups/virt/virtio-win/virtio-win.repo | sudo tee /etc/yum.repos.d/virtio-win.repo > /dev/null
sudo dnf install -y virtio-win
sudo cp /usr/share/virtio-win/virtio-win.iso /var/lib/libvirt/images/iso/
```

This is the more robust pattern generally — package-manager-delivered artifacts don't rot the way static download links do.

## 4. Isolated Lab Network

```bash
cat > /tmp/lab-net.xml <<'EOF'
<network>
  <name>lab-net</name>
  <bridge name='virbr-lab' stp='on' delay='0'/>
  <ip address='192.168.56.1' netmask='255.255.255.0'/>
</network>
EOF

sudo virsh net-define /tmp/lab-net.xml
sudo virsh net-start lab-net
sudo virsh net-autostart lab-net
```

Deliberately **no `<forward>` element** (no route to the internet on this network) and **no `<dhcp>` block** (no competing DHCP server) — DC01 serves DHCP for the domain itself later, and libvirt's dnsmasq handing out conflicting leases/DNS would silently break domain joins.

The DC gets **two NICs**:
- `lab-net` (VirtIO) — static IP, isolated lab LAN
- `default` (e1000e, libvirt's built-in NAT) — DHCP, internet access for eval activation and Windows Update

## 5. VM Creation

```bash
sudo virt-install \
  --name DC01 \
  --memory 4096 \
  --vcpus 2 \
  --cpu host-passthrough \
  --os-variant win2k22 \
  --boot uefi \
  --disk path=/var/lib/libvirt/images/DC01.qcow2,size=60,format=qcow2,bus=virtio \
  --cdrom /var/lib/libvirt/images/iso/SERVER_EVAL_x64FRE_en-us.iso \
  --disk path=/var/lib/libvirt/images/iso/virtio-win.iso,device=cdrom \
  --network network=lab-net,model=virtio \
  --network network=default,model=e1000e \
  --graphics spice \
  --video qxl
```

## 6. Windows Setup

- Edition: **Windows Server 2022 Standard Evaluation (Desktop Experience)** — Standard over Datacenter (no need for unlimited VM licensing in a single-DC lab); Desktop Experience over Core to keep Server Manager/ADUC/GPMC visible while learning.
- Product key screen → **"I don't have a product key"**
- Disk selection showed no drives (expected — no virtio storage driver yet) → **Load driver** → browsed to virtio-win CD → `viostor\2k22\amd64` → disk appeared → proceeded with install.
- Post-install: ran `virtio-win-guest-tools.exe` from the virtio-win CD to pick up the NIC driver, balloon driver, and QEMU guest agent (the network adapter was completely invisible in Windows until this ran).
- Network prompt ("allow this PC to be discoverable") → **Yes** — sets the Private network profile, which is required for AD DS/Windows Firewall to open the right ports later; Public profile blocks it silently.

## 7. Machine Identity & Static IP

```powershell
Rename-Computer -NewName "DC01" -Restart
```

```powershell
Get-NetAdapter | Format-Table Name, InterfaceDescription, MacAddress, Status
```

Identified the VirtIO adapter (`Ethernet 2`, "Red Hat VirtIO Ethernet Adapter") vs. the NAT adapter (`Ethernet`, "Intel(R) 82574L").

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet 2" -IPAddress 192.168.56.10 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet 2" -ServerAddresses 127.0.0.1
```

**Deliberately no `-DefaultGateway` on the lab adapter** — the NAT adapter already carries the default route via DHCP. Verified only one default route existed before and after the change:

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0"
```

Snapshot taken at this point as a clean rollback point before touching AD DS:

```bash
sudo virsh snapshot-create-as DC01 01-clean-install "Windows installed, renamed to DC01, static IP set on lab-net"
```
