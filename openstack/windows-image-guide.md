# OpenStack Windows Server Image Creation Guide

This guide walks you through building a Windows Server image (2019 or 2022) for OpenStack, complete with VirtIO drivers, Cloudbase-Init, and sysprep.

---

## Prerequisites

### Install Required Packages

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils \
    virt-manager libguestfs-tools genisoimage wget
```

---

## Step 1: Download ISOs

Create a working directory and store its path in a variable so every later command can
reference it with an absolute path:

```bash
export WORKDIR="$HOME/openstack-windows-image"
mkdir -p "$WORKDIR"
cd "$WORKDIR"
```

### VirtIO Drivers

```bash
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso \
    -O virtio-win.iso
```

### Windows Server ISO

> **Note:** Microsoft's evaluation download links change over time. If a link below
> fails, grab the current ISO from the
> [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/).

**Windows Server 2022 (Evaluation):**
```bash
wget -O WS2022-eval.iso \
    "https://go.microsoft.com/fwlink/p/?LinkID=2195280&clcid=0x409&culture=en-us&country=US"
```

**Windows Server 2019 (Evaluation):**
```bash
wget -O WS2019-eval.iso \
    "https://go.microsoft.com/fwlink/p/?LinkID=2195167&clcid=0x409&culture=en-us&country=US"
```

---

## Step 2: Create the Disk Image

```bash
qemu-img create -f qcow2 windows-server-2022.qcow2 50G
```

---

## Step 3: Prepare Cloudbase-Init Config

Create the config file:

```bash
vi cloudbase-init.conf
```

Paste the following content:

```ini
[DEFAULT]
username=Administrator
groups=Administrators
inject_user_password=false
first_logon_behaviour=no

config_drive_raw_hdd=true
config_drive_cdrom=true
config_drive_vfat=true

bsdtar_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\bin\bsdtar.exe
mtools_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\bin\

verbose=true
debug=true
logdir=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\log\
logfile=cloudbase-init.log
default_log_levels=comtypes=INFO,suds=INFO,iso8601=WARN,requests=WARN

mtu_use_dhcp_config=true
ntp_use_dhcp_config=true

local_scripts_path=C:\Program Files\Cloudbase Solutions\Cloudbase-Init\LocalScripts\

metadata_services=cloudbaseinit.metadata.services.configdrive.ConfigDriveService,cloudbaseinit.metadata.services.httpservice.HttpService

plugins=cloudbaseinit.plugins.common.mtu.MTUPlugin,cloudbaseinit.plugins.common.sethostname.SetHostNamePlugin,cloudbaseinit.plugins.windows.extendvolumes.ExtendVolumesPlugin,cloudbaseinit.plugins.common.userdata.UserDataPlugin

allow_reboot=false
stop_service_on_exit=false
check_latest_version=false
```

> **Why these settings matter:**
> - Setting `plugins=` **overrides** Cloudbase-Init's defaults — anything not listed is
>   disabled. This config intentionally leaves password handling out: the initial password
>   is set interactively during first-boot setup (OOBE) via the console, and later password
>   resets are handled live by the **QEMU Guest Agent** (see Step 12), so no Cloudbase-Init
>   password plugin is needed.
> - `config_drive_raw_hdd` (not `hhd`) — Cloudbase-Init silently ignores unknown keys, so
>   a typo here fails without any error.

### Build the Config ISO

```bash
genisoimage -o cloudbase-config.iso -J -r cloudbase-init.conf
```

---

## Step 4: Launch the VM

Launch the VM with `virt-install`, attaching all four disks (main disk, Windows ISO, Cloudbase config ISO, VirtIO drivers ISO).

> **Important:** libvirt resolves `--disk path=` against its own context, not your shell's
> current directory, so always pass **absolute paths**. We use the `$WORKDIR` variable from
> Step 1, which the shell expands to an absolute path before `virt-install` runs. (A bare
> filename or a `~` path will fail to attach.)

```bash
virt-install \
  --name ws2022-openstack \
  --ram 4096 \
  --vcpus 2 \
  --cpu host \
  --disk path=$WORKDIR/windows-server-2022.qcow2,format=qcow2,bus=virtio \
  --disk path=$WORKDIR/WS2022-eval.iso,device=cdrom,bus=sata \
  --disk path=$WORKDIR/cloudbase-config.iso,device=cdrom,bus=sata \
  --disk path=$WORKDIR/virtio-win.iso,device=cdrom,bus=sata \
  --network network=default,model=virtio \
  --os-variant win2k22 \
  --graphics vnc,listen=0.0.0.0 \
  --boot cdrom,hd \
  --noautoconsole
```

> **For Windows Server 2019:** use `--os-variant win2k19` and point the second `--disk` at
> `WS2019-eval.iso`.

### Connect via VNC

Find the VNC display port:

```bash
virsh list --all
virsh vncdisplay ws2022-openstack
```

> **Note:** If the output is `:1`, the VNC port is `5901`. Use a VNC client (like Remmina) to connect to `<host-ip>:5901`.

---

## Step 5: Windows Installation (Detailed)

After connecting via VNC, follow these steps to install Windows Server:

### 5.1 Initial Setup

1. **Language selection:** Choose `English` → Click `Next`
2. Click **`Install now`**

### 5.2 Operating System Selection

Select the following:
- **Operating System:** `Windows Server 2022 Standard Evaluation (Desktop Experience)` or `Windows Server 2019 Standard Evaluation (Desktop Experience)`
- **Architecture:** `x64`
- Click `Next`

### 5.3 License Terms

- Check `I accept the license terms`
- Click `Next`

### 5.4 Installation Type

- Select **`Custom: Install Windows only (advanced)`**

### 5.5 Load Storage Driver (Critical Step)

> The disk will NOT appear because VirtIO drivers are missing. You must load them.

1. Click **`Load driver`**
2. Click **`Browse`**
3. Navigate to the VirtIO ISO drive (labeled `virtio-win`)
4. Browse to: `viostor > 2k22 > amd64` (for Windows Server 2022)
   *For Windows Server 2019, use: `viostor > 2k19 > amd64`*
5. Select the folder and click `OK`
6. You should see: **`Red Hat VirtIO SCSI controller`**
7. Click `Next`

### 5.6 Complete Installation

- The disk will now appear (unallocated space)
- Select the disk and click `Next`
- Wait for Windows to copy files and complete installation (VM will reboot automatically)

---

## Step 6: Post-Installation Driver Setup (Inside Windows)

After logging into Windows (default Administrator account with no password — just press Enter):

### Install VirtIO Drivers (`virtio-win-gt-x64.msi`)

1. Open the VirtIO ISO drive in File Explorer
2. Find **`virtio-win-gt-x64.msi`** (the driver installer)
3. Double-click to install
4. Follow the installation wizard (default settings are fine)
5. **Restart if prompted** — this installs:
   - Network driver (NIC will work)
   - Balloon driver (memory management)
   - Serial driver (console support)

### Install Guest Tools (`virtio-win-guest-tools.exe`)

1. On the same VirtIO ISO drive, find **`virtio-win-guest-tools.exe`**
2. Double-click to install and follow the wizard (default settings are fine)
3. **Restart if prompted**

> This all-in-one installer adds the **QEMU Guest Agent** (`qemu-ga`) on top of the VirtIO
> drivers. The guest agent is required later for the live password reset via
> `openstack server set --password` (Step 12), so don't skip it.

### Verify Installation

After reboot, check Device Manager — no yellow warning signs should remain. You can also
confirm the guest agent is present:

```powershell
Get-Service QEMU-GA
```

---

## Step 7: Install Cloudbase-Init

### 7.1 Download Cloudbase-Init

Open a browser (Edge will work) and go to:
```
https://github.com/cloudbase/cloudbase-init/releases
```

Download the latest stable `.msi` file (e.g., `CloudbaseInitSetup_x64.msi`)

### 7.2 Installation

1. Run the installer
2. On the **Configuration options** screen:
   - Change **Username** from `admin` to **`Administrator`**
   - Leave other defaults
3. Complete the installation

> **Do NOT check Sysprep or Shutdown yet** — we will configure additional settings first.

### 7.3 Update Cloudbase-Init Configuration

After installation, replace the config file at:
```
C:\Program Files\Cloudbase Solutions\Cloudbase-Init\conf\cloudbase-init.conf
```

Use the same configuration content from `cloudbase-config.iso` (created in Step 3). The
config ISO is mounted as a CD-ROM drive inside the VM — copy the `.conf` file from there.

---

## Step 8: PowerShell Configuration

Open **PowerShell as Administrator** and run the following:

### Enable Remote Desktop

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
    -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Start-Service TermService
Set-Service -Name TermService -StartupType Automatic
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
    -Name 'UserAuthentication' -Value 1
```

### Disable the Firewall

In OpenStack, network traffic is filtered at the infrastructure layer by **security groups**,
so the in-guest Windows firewall is redundant. Disabling it avoids managing two separate
firewalls and prevents connectivity surprises after deployment:

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
Get-NetFirewallProfile | Select-Object Name, Enabled
```

### Verify Administrator Account

```powershell
net user Administrator
# The account should show as Active
```

### Verify RDP is Listening

```powershell
netstat -an | findstr 3389
# Expected: TCP 0.0.0.0:3389 ... LISTENING
```

### Configure QEMU Guest Agent

```powershell
Get-Service QEMU-GA
Set-Service -Name qemu-ga -StartupType Automatic
Start-Service -Name qemu-ga
```

### Verify Cloudbase-Init Service

```powershell
Get-Service cloudbase-init
```

---

## Step 9: Extend the System Volume

> **Note:** This step is only required for **Windows Server 2022**. Windows Server 2019 does not have this issue and can skip to Step 10.

> This step removes the recovery partition so the system volume can be extended, which is necessary for OpenStack to correctly resize the disk on deployment.

Open `diskpart` in PowerShell:

```powershell
diskpart
```

Then run:

```
list volume
select volume X        # Replace X with the volume number to delete (e.g. recovery partition)
delete volume override

select volume Y        # Replace Y with the volume number of drive C:
extend

exit
```

---

## Step 10: Final Sysprep and Shutdown

> ⚠️ **CRITICAL:** After running Sysprep, DO NOT boot the VM again. Boot only after the image conversion is complete.

### Run Sysprep

1. Open **Cloudbase-Init** installer again (or run Sysprep manually)
2. On the final screen, check BOTH:
   - ✅ **Run Sysprep**
   - ✅ **Shutdown**
3. Click **Finish**

The VM will:
- Run Sysprep (generalize the Windows installation)
- Shut down automatically
- Become ready for image conversion

> **Important:** Do NOT power on the VM after this step. Proceed directly to Step 11.

---

## Step 11: Convert and Compress the Final Image

Once the VM is shut down, convert the raw qcow2 to a compressed, upload-ready image:

```bash
# For Windows Server 2022
qemu-img convert -c -O qcow2 windows-server-2022.qcow2 windows-server-2022-Ready.qcow2 -p

# For Windows Server 2019
qemu-img convert -c -O qcow2 windows-server-2019.qcow2 windows-server-2019-Ready.qcow2 -p
```

The `-p` flag shows progress. The resulting `*-Ready.qcow2` file is ready to upload to OpenStack Glance.

---

## Step 12: Upload to OpenStack and Create Server

### Upload Image to Glance

```bash
# Source your OpenStack RC file
source your-openstack-rc-file.sh

# Upload the image
openstack image create "Windows-Server-2022" \
  --file windows-server-2022-Ready.qcow2 \
  --disk-format qcow2 \
  --container-format bare \
  --property os_type=windows \
  --property hw_qemu_guest_agent=yes
```

> The `hw_qemu_guest_agent=yes` property tells Nova to attach the QEMU Guest Agent channel
> to instances built from this image. That channel is what makes the live password reset in
> the next section work. (If your cloud already sets this globally, you can omit it.)

### Create an Instance

```bash
openstack server create \
  --image Windows-Server-2022 \
  --flavor FLAVOR_ID \
  --network public \
  "my-windows-server"
```

### Set Administrator Password

```bash
openstack server set --password YOUR_PASSWORD SERVER_ID
```

> The initial Administrator password is set during first-boot setup on the console. To
> reset it later (e.g. if the customer forgets it), this command works through the
> **QEMU Guest Agent** running inside the guest — it injects the new password live, with no
> reboot required. This relies on `qemu-ga` being installed (Step 6/8) and the
> `hw_qemu_guest_agent=yes` image property set above.

### Access the Server

1. Open VNC console to complete first-time setup:
   ```bash
   openstack console url show SERVER_ID
   ```
2. Login with the password you set
3. After login, you can use RDP to connect to the server's IP address on port 3389


