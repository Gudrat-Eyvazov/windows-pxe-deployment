# PXE Windows Deployment

Network-boot UEFI computers into WinPE, capture a reference Windows installation, and deploy it to other machines, all from a single Ubuntu server.

**Stack:** Kea DHCP · iPXE + wimboot · nginx · WinPE · Samba · DISM

---

## Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Environment](#environment)
4. [Setup](#setup)
   1. [Server network](#1-server-network)
   2. [DHCP (Kea)](#2-dhcp-kea)
   3. [WinPE files](#3-winpe-files)
   4. [HTTP boot (iPXE, wimboot, nginx)](#4-http-boot-ipxe-wimboot-nginx)
   5. [Network driver for WinPE](#5-network-driver-for-winpe)
   6. [Image share (Samba)](#6-image-share-samba)
   7. [Capture a backup image](#7-capture-a-backup-image)
   8. [Prepare and capture the deployment image](#8-prepare-and-capture-the-deployment-image)
   9. [Deploy to other computers](#9-deploy-to-other-computers)
5. [Troubleshooting](#troubleshooting)
6. [Security notes](#security-notes)
7. [License](#license)

---

## Overview

### Goal

Install or restore Windows on many computers without USB sticks. A computer is connected to the lab network, powered on, boots from the network into WinPE, and from there it can:

- **capture** its Windows installation into a `.wim` image stored on the server, or
- **deploy** a stored image to its disk and boot into the freshly written Windows.

### Scope

- UEFI clients only (Legacy BIOS boot is not covered).
- IPv4 only.
- Wired Ethernet on the clients (PXE over Wi-Fi is not possible).
- The PXE network is isolated, so the server can run its own DHCP service.

## Architecture

```
 Client (UEFI PXE)
      |
      | 1. DHCP request
      v
 Kea DHCP --- "boot file = ipxe.efi, server = 192.168.50.20" --->  client
      |
      | 2. TFTP: ipxe.efi (small file that firmware can fetch)
      v
 tftpd-hpa
      |
      | 3. iPXE starts, asks DHCP again, receives http://192.168.50.20/boot.ipxe
      v
 nginx (HTTP) <-- boot.ipxe, wimboot, bootmgfw.efi, BCD, boot.sdi, boot.wim
      |
      | 4. wimboot loads boot.wim into RAM and starts WinPE
      v
 WinPE (RAM disk X:)
      |
      | 5. net use Z: \\192.168.50.20\images
      v
 Samba share /srv/images <--> dism /Capture-Image | dism /Apply-Image
```

All boot services listen on the server's PXE adapter. The second adapter provides internet access for package installation.

| Component | Role | Why it is used |
|---|---|---|
| **Kea DHCP** | Assigns IP addresses and tells PXE clients what to boot | Client classes allow different boot files for firmware PXE and for iPXE |
| **tftpd-hpa** | Serves `ipxe.efi` | PXE firmware only speaks TFTP, so one small file is served this way |
| **iPXE** | Second-stage network bootloader | Speaks HTTP, which firmware PXE does not |
| **wimboot** | Boots a Windows `.wim` from iPXE | Loads `boot.wim` into RAM and starts the Windows boot manager |
| **nginx** | Serves boot files over HTTP | Large files (`boot.wim` is several hundred MB) transfer much faster and more reliably than over TFTP |
| **WinPE** | Minimal Windows environment | Provides `diskpart`, `dism`, `bcdboot` and an SMB client |
| **Samba** | File share for images | WinPE can write and read multi-gigabyte images over SMB |
| **DISM** | Captures and applies `.wim` images | Built into WinPE, supports compression |

## Environment

The values below are examples. Replace them with your own.

| Item | Example value |
|---|---|
| PXE server | Ubuntu Server with two network adapters |
| Internet adapter | `ens33` = `192.168.204.131/24`, gateway `192.168.204.2` |
| PXE adapter | `ens37` = `192.168.50.20/24` |
| DHCP pool | `192.168.50.100` - `192.168.50.200` |
| Image share | `\\192.168.50.20\images` (`/srv/images` on the server) |
| Share user | `pxeuser` |
| Boot files directory | `/var/lib/tftpboot` |

Requirements:

- Ubuntu Server with **two network adapters**: one for internet access, one on the isolated PXE network.
- A Windows computer with the **Windows ADK** and the **WinPE add-on** to build WinPE.
- Enough free disk on the server for images. Captured images were 16 GB to 36 GB each in our lab, so plan **100 GB or more**.

---

## Setup

### 1. Server network

The DHCP server cannot obtain its own address from DHCP, so the PXE adapter gets a static IP. The internet adapter also gets a static address with a default route and DNS servers.

```bash
cp -a /etc/netplan /root/netplan-backup
nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.204.131/24
      routes:
        - to: default
          via: 192.168.204.2
      nameservers:
        addresses: [192.168.204.2, 1.1.1.1]
    ens37:
      dhcp4: false
      addresses:
        - 192.168.50.20/24
```

```bash
chmod 600 /etc/netplan/*.yaml
netplan generate
netplan apply
```

Use the adapter names of your own server (`ip link`). Only the PXE adapter has no default route, so internet traffic always leaves through the other adapter.

### 2. DHCP (Kea)

```bash
apt update
apt install kea-dhcp4-server -y
nano /etc/kea/kea-dhcp4.conf
```

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": [ "ens37" ] },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/run/kea/kea4-ctrl-socket"
    },
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "valid-lifetime": 3600,

    "client-classes": [
      {
        "name": "iPXE",
        "test": "substring(option[77].hex,0,4) == 'iPXE'",
        "boot-file-name": "http://192.168.50.20/boot.ipxe"
      },
      {
        "name": "UEFI-first-stage",
        "test": "not (substring(option[77].hex,0,4) == 'iPXE') and option[93].hex == 0x0007",
        "boot-file-name": "ipxe.efi"
      }
    ],

    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.50.0/24",
        "next-server": "192.168.50.20",
        "pools": [ { "pool": "192.168.50.100 - 192.168.50.200" } ],
        "option-data": [ { "name": "subnet-mask", "data": "255.255.255.0" } ]
      }
    ]
  }
}
```

```bash
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf      # validate the configuration
systemctl restart kea-dhcp4-server
```

**Two client classes, two boot stages.** The firmware's PXE client asks DHCP first and receives `ipxe.efi`. After iPXE starts, it asks DHCP again and identifies itself with user-class `iPXE` (option 77). That second request receives the URL of the iPXE script. Without the split, iPXE would be handed `ipxe.efi` again and loop. Option 93 value `0x0007` means "UEFI x64 client".

### 3. WinPE files

**On a Windows computer** with the ADK and WinPE add-on installed, open the *Deployment and Imaging Tools Environment* as Administrator and build the ISO:

```bat
copype amd64 C:\WinPE_amd64
MakeWinPEMedia /ISO C:\WinPE_amd64 C:\WinPE_amd64.iso
```

Copy it to the server:

```powershell
scp C:\WinPE_amd64.iso <user>@192.168.50.20:/home/<user>/
```

**On the server**, mount the ISO and copy its contents to the boot files directory:

```bash
mkdir -p /mnt/winpe-iso
mount -o loop /home/<user>/WinPE_amd64.iso /mnt/winpe-iso

apt install tftpd-hpa -y
mkdir -p /var/lib/tftpboot
cp -r /mnt/winpe-iso/. /var/lib/tftpboot/
systemctl restart tftpd-hpa
```

Mounting is only needed to read the ISO contents. The files used later are `EFI/Microsoft/Boot/bootmgfw.efi`, `Boot/BCD`, `Boot/boot.sdi` and `sources/boot.wim`.

Make sure TFTP serves the directory used above. The default root on some Ubuntu releases is `/srv/tftp`, so check `/etc/default/tftpd-hpa`:

```
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

```bash
systemctl restart tftpd-hpa
```

### 4. HTTP boot (iPXE, wimboot, nginx)

TFTP only delivers the small `ipxe.efi`. Everything large is delivered over HTTP.

```bash
apt install -y nginx ipxe
cp /usr/lib/ipxe/ipxe.efi /var/lib/tftpboot/ipxe.efi
wget -O /var/lib/tftpboot/wimboot \
  https://github.com/ipxe/wimboot/releases/latest/download/wimboot
```

- `ipxe.efi` is the first file a network-booting computer reads.
- `wimboot` is the loader that lets iPXE boot a Windows `.wim` (WinPE) image over the network.

Serve the same directory over HTTP, bound to the PXE adapter:

```bash
cat > /etc/nginx/sites-available/pxe <<'EOF'
server {
    listen 192.168.50.20:80;
    root /var/lib/tftpboot;
}
EOF
ln -s /etc/nginx/sites-available/pxe /etc/nginx/sites-enabled/pxe
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl restart nginx
curl -I http://192.168.50.20/sources/boot.wim
```

- `listen` accepts requests only on the PXE address.
- `root` makes the files in `/var/lib/tftpboot` visible over HTTP.
- The `ln -s` line enables the new site, and removing `default` removes the stock nginx page.
- `curl` should return `200 OK` and the real size of `boot.wim`.

Create the iPXE script `/var/lib/tftpboot/boot.ipxe`. It tells the client how to boot:

```
#!ipxe
kernel http://192.168.50.20/wimboot
initrd http://192.168.50.20/EFI/Microsoft/Boot/bootmgfw.efi bootmgfw.efi
initrd http://192.168.50.20/Boot/BCD BCD
initrd http://192.168.50.20/Boot/boot.sdi boot.sdi
initrd http://192.168.50.20/sources/boot.wim boot.wim
boot
```

`wimboot` builds a virtual boot volume in RAM from the files listed as `initrd` and starts `bootmgfw.efi`.

To watch a client boot:

```bash
tail -f /var/log/nginx/access.log
```

The requests should appear in order: `boot.ipxe`, `wimboot`, `bootmgfw.efi`, `BCD`, `boot.sdi`, `boot.wim`, all with status `200`.

> **Secure Boot:** the Ubuntu `ipxe.efi` binary is not signed by a key that client firmware trusts. If Secure Boot is enabled on the client, disable it for this setup.

### 5. Network driver for WinPE

PXE itself works without any Windows driver, because the firmware has its own network stack. Once WinPE is running, **Windows needs a driver for the client's network adapter**. A default WinPE image does not contain drivers for every computer, and without one `ipconfig` shows no adapter and the share cannot be reached.

The driver must therefore be added to `boot.wim`, once for each network adapter model you plan to boot.

#### What you need

A driver package for the **client's wired network adapter**, as an extracted folder containing `.inf`, `.sys` and `.cat` files (an installer `.exe` will not work). To find the right one:

- On a working Windows installation of the same computer model, open Device Manager, then *Network adapters*, the adapter, *Properties*, *Details*, *Hardware Ids*. The `VEN_xxxx&DEV_xxxx` values identify the adapter.
- Many vendors (Dell, HP, Lenovo) publish a *WinPE driver pack* for their models. Otherwise download the driver from the adapter vendor.
- In a running WinPE, `pnputil /enum-devices /problem` lists devices that have no driver.

Place the extracted driver folder on the Windows computer used for image servicing, for example `C:\drivers\eth`. In our lab the adapter was an Intel I219 and the driver file was `e1d.inf`.

#### Test it first

Before modifying `boot.wim`, confirm the driver works. Copy the driver folder to a USB stick, boot WinPE, and run:

```
drvload E:\drivers\eth\e1d.inf
wpeinit
ipconfig
```

If an IPv4 address appears, the driver is correct. (If WinPE does not assign a drive letter to the stick, use `diskpart`, `select volume N`, `assign letter=E`.)

#### Add the driver to boot.wim

Do this on a **full Windows computer**, in an elevated PowerShell, not inside WinPE. Replace `C:\drivers\eth` with the path of your driver folder:

```powershell
mkdir C:\wim, C:\mount
scp <user>@192.168.50.20:/var/lib/tftpboot/sources/boot.wim C:\wim\boot.wim
dism /Get-WimInfo /WimFile:C:\wim\boot.wim
dism /Mount-Image /ImageFile:C:\wim\boot.wim /Index:1 /MountDir:C:\mount
dism /Image:C:\mount /Add-Driver /Driver:C:\drivers\eth /Recurse
dism /Image:C:\mount /Get-Drivers
```

#### Find the published driver name (the "oem" name)

When DISM adds a driver to an image, it stores it under a generic name of the form `oemN.inf`. The original file name from the driver folder (for example `e1d.inf`) is kept only as metadata. `/Get-Drivers` lists both:

```
Published Name : oem0.inf
Original File Name : e1d.inf
```

The **Published Name** (`oem0.inf` here) is the file that exists inside the image at `X:\Windows\INF\`. `X:` is the WinPE RAM disk. The number can be different in your image, so always read it from the `/Get-Drivers` output. Do not use the file name from your driver folder.

#### Load the driver at startup

WinPE runs `startnet.cmd` when it starts. `wpeinit` initializes networking, and `drvload` loads a driver. In our tests the adapter was not bound automatically even though the driver was in the image, so the driver is loaded explicitly, **before** `wpeinit`:

```powershell
Set-Content -Path C:\mount\Windows\System32\startnet.cmd `
  -Value "drvload X:\Windows\INF\oem0.inf`r`nwpeinit" -Encoding ASCII
type C:\mount\Windows\System32\startnet.cmd
dism /Unmount-Image /MountDir:C:\mount /Commit
```

Replace `oem0.inf` with the Published Name from the previous step. The resulting `startnet.cmd` contains two lines:

```
drvload X:\Windows\INF\oem0.inf
wpeinit
```

#### Publish the updated image

Copy the file to the server, back up the old one, and replace it:

```powershell
scp C:\wim\boot.wim <user>@192.168.50.20:/home/<user>/boot.wim
```

```bash
cp /var/lib/tftpboot/sources/boot.wim /root/boot.wim.orig      # backup
cp /home/<user>/boot.wim /var/lib/tftpboot/sources/boot.wim
```

nginx serves the new file immediately. Boot a client and run `ipconfig` in WinPE: an address from the DHCP pool should be present.

### 6. Image share (Samba)

WinPE writes captured images to the server and reads them back for deployment over SMB.

```bash
apt install -y samba
mkdir -p /srv/images
adduser --no-create-home --disabled-password --shell /usr/sbin/nologin pxeuser
chown pxeuser /srv/images
smbpasswd -a pxeuser
```

In `/etc/samba/smb.conf`, add to the `[global]` section so the share is only visible on the PXE adapter:

```
   interfaces = ens37 lo
   bind interfaces only = yes
```

Add at the end of the file:

```
[images]
   path = /srv/images
   read only = no
   valid users = pxeuser
```

```bash
systemctl restart smbd
```

Images are large, so check free space before each capture:

```bash
df -h /
du -sh /srv/images/*
```

### 7. Capture a backup image

Capture the reference computer **before** running Sysprep. Sysprep cannot be undone, and this image is the way back if it fails.

Capture from **WinPE**, never from the running Windows. Boot the reference computer over the network (F12 > Network Boot), then connect to the share and find the Windows volume. Drive letters in WinPE are not predictable:

```
net use Z: \\192.168.50.20\images /user:pxeuser
dir Z:

diskpart
list volume
exit
dir D:\Windows
```

If the Windows volume has no letter, run `select volume N` and `assign letter=D` in `diskpart`. Then capture:

```
dism /Capture-Image /ImageFile:Z:\pc-clean.wim /CaptureDir:D:\ /Name:"PC-Clean" /Compress:fast
dism /Get-WimInfo /WimFile:Z:\pc-clean.wim
```

Wait for `The operation completed successfully` and do not power off during the capture. `/Compress:fast` is quicker; `/Compress:max` produces smaller files but is much slower.

### 8. Prepare and capture the deployment image

An image that will be deployed to **several** computers must be generalized with Sysprep. Otherwise all copies share the same computer name and SID and carry the source machine's hardware state. (An image restored to the same computer does not need Sysprep.)

1. In Windows on the reference computer, run Sysprep from the GUI:

   ```
   C:\Windows\System32\Sysprep\sysprep.exe
   ```

   Select **Enter System Out-of-Box Experience (OOBE)**, tick **Generalize**, set Shutdown Options to **Shutdown**, and click OK.
2. After the computer shuts down, **do not boot Windows**. Doing so starts OOBE and consumes the generalized state. Power it on and boot over the network into WinPE (F12 > Network Boot).
3. In WinPE, find the Windows volume as before (`diskpart`, `list volume`, `exit`, `dir D:\Windows`) and confirm that Sysprep succeeded:

   ```
   dir D:\Windows\System32\Sysprep\Sysprep_succeeded.tag
   ```

   If the file exists, Sysprep finished successfully.
4. Capture the generalized image:

   ```
   net use Z: \\192.168.50.20\images /user:pxeuser
   dism /Capture-Image /ImageFile:Z:\win-golden.wim /CaptureDir:D:\ /Name:"Win-Golden" /Compress:fast
   dism /Get-WimInfo /WimFile:Z:\win-golden.wim
   ```
5. Leave WinPE with `wpeutil shutdown`.

If Sysprep fails, read `C:\Windows\System32\Sysprep\Panther\setupact.log` and `setuperr.log` before trying again (see [Troubleshooting](#troubleshooting)). The registry value `GeneralizationState` is not a reliable success indicator, so rely on `Sysprep_succeeded.tag`.

### 9. Deploy to other computers

Create the partition layout `/srv/images/uefi.txt`:

```
select disk 0
clean
convert gpt
create partition efi size=260
format quick fs=fat32 label="System"
assign letter=S
create partition msr size=16
create partition primary
format quick fs=ntfs label="Windows"
assign letter=W
exit
```

Create the deployment script `/srv/images/deploy.cmd`. The image file name is a parameter (`%1`), so one script serves every image:

```bat
@echo off
echo.
echo WARNING: Disk 0 will be COMPLETELY ERASED!
set /p ok=Type YES to continue: 
if /i not "%ok%"=="YES" exit /b
diskpart /s Z:\uefi.txt
dism /Apply-Image /ImageFile:Z:\%1 /Index:1 /ApplyDir:W:\
bcdboot W:\Windows /s S: /f UEFI
wpeutil reboot
```

Files created on Linux need Windows line endings (CRLF) for `cmd`. Run this **once** per file (running it again adds a second `\r`):

```bash
sed -i 's/$/\r/' /srv/images/deploy.cmd /srv/images/uefi.txt
chown pxeuser /srv/images/*
```

On the target computer:

1. Boot over the network (F12 > Network Boot) and confirm that `ipconfig` shows an address.
2. Verify the disk. `Disk 0` must be the internal disk, not a USB stick:

   ```
   diskpart
   list disk
   exit
   ```
3. Run the deployment:

   ```
   net use Z: \\192.168.50.20\images /user:pxeuser
   Z:\deploy.cmd win-golden.wim
   ```
4. Type `YES`. The disk is erased, the image is written, `bcdboot` creates the boot entry, and the computer restarts.
5. Put **Windows Boot Manager** first in the firmware boot order so the computer does not return to PXE.

`bcdboot` is what makes the computer bootable: `dism /Apply-Image` writes files only, not boot records.

On first boot, a computer deployed from a generalized image shows the Windows setup (OOBE): region selection, then account creation. This is the expected result and confirms that Sysprep worked.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| WinPE starts, `ipconfig` shows no adapter | No driver for the NIC in `boot.wim`, or the wrong `oem` name in `startnet.cmd` | Add the driver and load it with its Published Name ([section 5](#5-network-driver-for-winpe)) |
| `Error: 3` / "cannot find the path" during capture | `Z:` is not mounted in this session, or the Windows volume is not `D:` | Run `net use Z:` again and verify with `dir D:\Windows` |
| `0xc1420127` already mounted | Stale DISM mount | `dism /Unmount-Image /MountDir:C:\mount /Discard`, then `dism /Cleanup-Wim` |
| Sysprep stops with `0x800F0975` | Reserved storage in use | See below |
| Sysprep `0x8007001f` | Invalid state left by earlier failed attempts | Restore the computer from the backup image |
| Black screen after deployment | Image captured from a machine whose Sysprep had failed | Capture again from a correctly generalized machine |
| Capture stops with share errors | Server disk is full | Check `df -h /`, free space or extend the volume |

### Sysprep stops with `0x800F0975`

`setupact.log` shows `Audit mode cannot be turned on if reserved storage is in use`. Sysprep stopped at validation, so nothing was damaged. Install pending Windows updates, **restart** (not shut down), wait 10 to 15 minutes for background servicing to finish, then disable reserved storage:

```powershell
DISM /Online /Get-ReservedStorageState
DISM /Online /Set-ReservedStorageState /State:Disabled
DISM /Online /Get-ReservedStorageState
```

The last command should report that reserved storage is disabled. If the disable command fails with the same error, servicing is still pending: restart, wait, and repeat. It can be re-enabled later with `/State:Enabled`. Then run Sysprep again.

## Security notes

- Keep the PXE network **isolated**. A second DHCP server on a production network hands clients wrong boot settings.
- `deploy.cmd` **erases disk 0** after `YES` is typed. Always check `list disk` first, and do not run it automatically from `startnet.cmd`.
- The Samba share is bound to the PXE adapter and requires a password. Do not store the password in scripts; `net use` prompts for it if it is omitted.
- Disabling Secure Boot lowers client security. Re-enable it on production machines after deployment.
- Images contain whatever was on the reference machine (accounts, saved credentials, software). Build reference images from a clean installation.

## License

Add a license of your choice (for example MIT) as a `LICENSE` file. Windows, the Windows ADK, DISM and Sysprep are Microsoft products and are subject to Microsoft's license terms; deploying Windows images requires valid Windows licenses.
