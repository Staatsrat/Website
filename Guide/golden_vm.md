## Windows Download

You will need **Windows 11 Enterprise**.

You can download it for free from the official Microsoft site using this link:

```
https://www.microsoft.com/en-us/evalcenter/download-windows-11-enterprise
```

(This is a 90-day free license.)

---

## Ventoy

You will need Ventoy as a way to boot from the USB stick. Install it using:

```
wget https://github.com/ventoy/Ventoy/releases/download/v1.1.05/ventoy-1.1.05-linux.tar.gz
tar -xzf ventoy-*-linux.tar.gz
cd ventoy-1.1.05
```

Install it to the **unmounted** USB stick using:

```
sudo ./Ventoy2Disk.sh -i /dev/sda
```

You should see something like: `Install Ventoy to /dev/sda successfully finished.`

Mount the USB stick again:

```
lsblk -o NAME,SIZE,MOUNTPOINTS,LABEL
```

Then execute:

```
sudo mkdir -p /mnt/ventoy
sudo mount /dev/sda1 /mnt/ventoy
```

Verify:

```
ls /mnt/ventoy
```

Copy the ISO to the USB stick:

```
sudo cp ~/Downloads/windows.iso /mnt/ventoy/
```

(I called my Windows ISO `windows.iso`, yours might be called something different.)

Then:

```
sync
```

Check again:

```
ls -lh /mnt/ventoy/
```

Unmount the USB stick:

```
sync
sudo umount /mnt/ventoy
```

Verify:

```
lsblk -o NAME,SIZE,MOUNTPOINTS,LABEL
```

---

## Boot

1. Now plug the USB stick into your second PC. Spam **F12** to enter the boot menu, choose the Ventoy stick, and choose the Windows ISO — that should be the only option on the USB stick. Then do the installation. You will figure that out. I would disable all cloud and tracking services.

2. After that, boot into Windows and enable **Hyper-V** and the **Hyper-V Manager**. Press `Win + R` and type `OptionalFeatures.exe`, then check **Hyper-V** and **Windows Hypervisor Platform**. After that, restart the PC.

3. Then create an 80 GB volume for the Windows VM. Open PowerShell as Administrator:

```
diskpart
create vdisk file="C:\VHDs\WinBase.vhdx" maximum=81920 type=expandable
select vdisk file="C:\VHDs\WinBase.vhdx"
attach vdisk
create partition primary
format fs=ntfs quick label="WinBase"
assign letter=V
exit
```

```
diskpart
list volume
exit
```

There should be a volume about 80 GB with the letter `V` and the label `WinBase`.

After that, search for the **Hyper-V Manager** program and open it. Create a new VM. For the ISO, plug in the Ventoy USB stick, open it in Explorer, and mount the `windows.iso` in the folder. That should create a DVD drive. Then continue creating the VM and choose the DVD file as the ISO.

4. Now it should boot into the Windows installation.

If Windows complains during the installation because the system requirements are not met, type these commands by pressing `Shift + F10` to avoid it. Do that **before** it tells you your system does not meet the requirements, because then it is already too late — so do it when it asks for the keyboard layout:

```
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassTPMCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassSecureBootCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassRAMCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassCPUCheck /t REG_DWORD /d 1 /f
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassStorageCheck /t REG_DWORD /d 1 /f
```

Then choose the 80 GB drive if it asks you where to install it.

In the VM, install all the useful stuff, for example FLARE VM:

```
https://raw.githubusercontent.com/mandiant/flare-vm/main/install.ps1
```

Download the `install.ps1` file to your desktop and make sure your PC will allow the execution:

```
Set-ExecutionPolicy Unrestricted -Scope Process
```

Then execute the file:

```
cd $([Environment]::GetFolderPath("Desktop"))
.\install.ps1
```

After you have installed all the tools you need, shut down the VM.

---

## Create a Checkpoint (the "Golden Image")

**Important:** The checkpoint logic is different from what you wrote down. You create a checkpoint **once**, and later you apply it to reset.

1. Shut down the VM cleanly.
2. In Hyper-V Manager: right-click the VM → **Checkpoint** (or "Snapshot").
3. Rename the checkpoint to something like `Clean_Base`.

This is now your clean baseline with all tools installed.

---

## Start an Analysis Session

**Before running any malware:**

- **Isolate the network!** In the VM settings, set the network adapter to **"Not connected"**. Or use an internal switch that is only connected to a separate FakeNet-NG VM. **Never** leave the "Default Switch" active — it has NAT and therefore access to your real network. Otherwise the malware can infect your entire network!
- **Disable Windows Defender.** Otherwise your analysis tools will be constantly deleted.
- **Malware transfer:** Think about how to get samples into the VM without touching the host. For example, burn an ISO with the samples and mount it in the VM. **No** shared folders, no clipboard sharing.

Then start the VM (right-click → **Start**) and perform your analysis.

---

## Reset After Analysis

1. Shut down the VM.
2. In Hyper-V Manager: right-click the checkpoint `Clean_Base` → **Apply**.
3. Confirm → the VM is back to a clean state.
4. Start the VM again — done.

You can manage multiple checkpoints, but for the beginning, `Clean_Base` is enough.

---

## Important Security Rules

- **Network isolation is mandatory**, not optional. This is the most important point.
- **Secure the host:** On the host (PC2, maintenance Windows), keep Windows Firewall on, no shares, no remote access. The host should **not** be on the same network as PC1 while malware is running.
- **Windows Defender in the VM off**, otherwise tools will be deleted.
- **No shared folders, no clipboard sharing, no USB passthrough** into the VM.
- **Documentation:** Keep track of which checkpoint is which, which sample you analyzed when, and which IOCs you found

---

Now have fun with whatever you want to do. But be careful — we haven't done any network segmentation yet, so a malicious program could still infect your whole network!
