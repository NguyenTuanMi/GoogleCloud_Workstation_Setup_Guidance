# GoogleCloud_Workstation_Setup_Guidance

This is a guidance on how to set up google cloud linux workstation with GPU and remote using Sunshine+Moonlight+Tailscale and deploy NTU Mecatron software stack on this workstation. 

## 1. Create a Google Cloud account

To create a workstation, first you have to sign up for a Gmail account and use that Gmail to create a [Google Cloud](https://cloud.google.com/?e=48754805&utm_source=google&utm_medium=cpc&utm_campaign=Cloud-SS-DR-GCP-1713664-GCP-DR-APAC-TW-zh-Google-BKWS-MIX-GenericCloud&utm_content=c-Hybrid+%7C+BKWS+-+BRO+%7C+Txt+-+Generic+Cloud-Cloud+Generic-Core+GCP-TW_en-6458750523&utm_term=google%20cloud&gclsrc=aw.ds&gad_source=1&gad_campaignid=19506976549&gclid=Cj0KCQjwnbrUBhDOARIsAKKhPpcEr2HJoKauwqwlADN-gvgZCSfG1-QNP1aUzgCUpNGQoQ443WOVBFoaAn0YEALw_wcB). 

For first time user, you will get 300 USD dollar in credits (which is a lot).

## 2. Create First Project

After getting inside the Google Cloud, you have to create a project (or it is automatially created on first time sign in). Then you have to navigate to the Cloud Shell, basically another terminal, to proceed with our setup. You can open this by clicking the shell icon on top right corner of the window, or you can just click G + S. 

When creating a workstation with GPU capability, we need to care about what type of machines we are going to use, how much persistent disk storage we are going to allocated, and the OS.

- To check the list of registered server regions and what types of machines available for each regions, please follow this [link](https://docs.cloud.google.com/compute/docs/regions-zones#available). 
- To check the list of machines, please follow this [link](https://docs.cloud.google.com/compute/docs/machine-resource).
- Google Cloud supports Ubuntu 22.04 and 24.04 and Window 10.

For this guidance, I use g2-standard-8 (8 vCPUs, 32 GB Memory), 1 x NVIDIA L4 Virtual Workstation, 100 GB persistent storage, and OS ubuntu-2204-jammy. You can adapt this to your purpose but issues arising from doing so are unknown and unchecked. 

### 2.1. Create a Workstation Instance

Firstly, you have to check what are the available computing servers. Ideally pick ones close to you physically. To do so, type this in the Cloud Shell:
```bash
gcloud compute accelerator-types list
```

If you already have in mind a specific region and you just want to see the available machines, type this: 
```bash
gcloud compute accelerator-types list --filter="zone:(your-zone-name)"
```
Example: 
```bash
gcloud compute accelerator-types list --filter="zone:(asia-southeast1-b)"
```

After finding your desired zone, config your compute/zone:
```bash
gcloud config set compute/zone your-zone-name
```

Create Workstation Instance. I name my workstation as test-workstation, but you can change if you want: 
```bash
gcloud compute instances create test-workstation     --zone=your-zone-name     --machine-type=g2-standard-8     --accelerator=type=nvidia-l4-vws,count=1     --maintenance-policy="TERMINATE"     --image-project=ubuntu-os-cloud     --image-family=ubuntu-2204-lts     --boot-disk-size=100     --boot-disk-type=pd-ssd     --network=default
```

If you create successfully, you can access the workstation using this cmd:
```bash
gcloud compute ssh test-workstation
```

#### 2.1.1 A possible issue

An issue you might face while creating a Google Cloud workstation instance:
```bash
 https://docs.cloud.google.com/compute/docs/virtual-workstation/linux-gpu

Error:

ERROR: (gcloud.compute.instances.create) Could not fetch resource:

 - Quota 'GPUS_ALL_REGIONS' exceeded.  Limit: 0.0 globally.

        metric name = compute.googleapis.com/gpus_all_regions

        limit name = GPUS-ALL-REGIONS-per-project

        limit = 0.0

        dimensions = global: global

Try your request in another zone, or view documentation on how to increase quotas: https://cloud.google.com/compute/quotas. 
```

#### 2.1.2. The cause

By default, new Google Cloud projects start with a GPU quota limit of 0.0 (both globally and regionally) to prevent accidental high-cost usage or abuse. To build your Linux virtual workstation using GPUs, you must manually request a quota increase. You have to upgrade your account from Free Trial to Full Account. Don't worry, Google Cloud will deduct your existing free credits first before touching your own money. 

#### 2.1.3. A solution

Follow these steps in the Google Cloud Console to request more quota:
- Open the Google Cloud Console.
- Navigate to IAM & Admin > Quotas (or search for "Quotas" in the top search bar).
- At the top filter bar, clear existing filters if needed, and look for the Filter box. Enter GPUS_ALL_REGIONS and press Enter.(Note: Depending on the specific guide you are following, you may also need to check for specific regional GPU metrics like NVIDIA_T4_GPUS or NVIDIA_L4_GPUS depending on what kind of GPU the workstation script requests).  
- Click on the checkbox next to the matching row (GPUS_ALL_REGIONS), then click Edit Quota (or click the three dots More actions -> Edit quota on the right side).
- In the Quota changes side pane, you enter your New value (e.g., 1 or 2 depending on how many GPUs your workstation configuration requires). For this case, just 1. 
- Fill in your email and a clear, professional justification (e.g., "Requesting GPU quota to build a Linux virtual workstation for graphical development/rendering workloads"). Providing a clear use case helps get requests processed smoothly. If you are a student, please state in this justification. 
- Click Submit request.

Approval time might be varied, but I get approved within 5 mins. 

### 2.2. Install base libraries inside the workstation

Update the software repositories:

```bash
sudo apt update
```

Install the base components: 
```bash
sudo apt install -y build-essential
sudo apt install -y libvulkan1
```

Update the gcc version for the NVIDIA driver:
```bash
sudo apt install -y gcc-12
sudo apt install -y linux-headers-$(uname -r)
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 12
sudo update-alternatives --config gcc
```

Installing NVDIA RTX Virtual Workstation driver and NVIDIA CUDA Toolkit:
```bash
curl -L https://storage.googleapis.com/compute-gpu-installation-us/installer/latest/cuda_installer.pyz --output cuda_installer.pyz
sudo python3 cuda_installer.pyz install_driver # This might restart your compute instance
sudo python3 cuda_installer.pyz install_cuda 
sudo reboot
```

### 2.3. Install the desktop environment

In my case, I use kubuntu-desktop instead of ubuntu-desktop, but you can choose whichever you want to use. Google said that kubuntu-desktop gives better rendering performance, so I stick with it:

```bash
sudo apt update
sudo apt -y install kubuntu-desktop
sudo apt -y install dialog
sudo reboot
```

### 2.4. Setup Tailscale on Google Cloud Virtual Machine

You have to setup tailscale on both your client laptop/PC and the Google Cloud Virtual Workstation in order for the two systems to communicate with each other. 

On Virtual Workstation (after you ssh in), type this cmd to the terminal:
```bash
curl -fsSL https://tailscale.com/install.sh | sh # Downloading and install tailscale
sudo tailscale up # This command will print a tailscale http link, you click on it and follow instructions to register your virtual workstation
```

Similarly for your laptop/PC. 

### 2.5. Setup Sunshine/Moonlight and prepare pinpairing

Sunshine must be setup on your host/server machine (in this case, its the virtual workstation), and moonlight must be installed on the client machines so you can remote desktop to Sunshine. 

On your laptop/PC, follow this guidance to install moonlight: 

On the Virtual Workstation, install Sunshine: 
```bash
wget https://github.com/LizardByte/Sunshine/releases/download/v0.23.1/sunshine-ubuntu-22.04-amd64.deb -O sunshine-ubuntu-amd64.deb
```

Enable and start Sunshine:
```bash
systemctl --user enable sunshine.service
systemctl --user start sunshine.service
systemctl --user status sunshine.service
```

#### 2.5.1. A possible issue

- Likely after pinpairing, you cannot view the desktop yet (see a black screen). 

#### 2.5.2. The cause

There is no X server running at all, right now. This is likely because the sddm (the display manager of kubuntu-desktop) is disabled/uninstalled/inactive/brooken. 

#### 2.5.3. A solution
First, you have to fact check the sddm status: 
```bash
systemctl status sddm
```

If you see something like this (likely):
```bash
○ sddm.service - Simple Desktop Display Manager
     Loaded: loaded (/lib/systemd/system/sddm.service; enabled; vendor preset: enabled)
     Active: inactive (dead)
       Docs: man:sddm(1)
             man:sddm.conf(5)
```

Then first you have to activate it:
```bash
sudo systemctl start sddm
```

Then you check again to see if it stayed up or died again: 
```bash
systemctl status sddm
```

If you see it on and encounter this issue or similar issue: 
```bash
● sddm.service - Simple Desktop Display Manager
     Loaded: loaded (/lib/systemd/system/sddm.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-08-25 02:59:03 UTC; 30s ago
       Docs: man:sddm(1)
             man:sddm.conf(5)
    Process: 29459 ExecStartPre=/bin/sh -c [ "$(cat /etc/X11/default-display-manager 2>/dev/null)" = "/usr/bin/sddm" ] (code=exited, status=0/SUCCESS)
   Main PID: 29461 (sddm)
      Tasks: 2 (limit: 38437)
     Memory: 57.0M
        CPU: 371ms
     CGroup: /system.slice/sddm.service
             └─29461 /usr/bin/sddm
Aug 25 02:59:03 test-workstation sddm[29461]: Initializing...
Aug 25 02:59:03 test-workstation sddm[29461]: Starting...
Aug 25 02:59:03 test-workstation sddm[29461]: Logind interface found
Aug 25 02:59:03 test-workstation sddm[29461]: Adding new display on vt 1 ...
Aug 25 02:59:03 test-workstation sddm[29461]: Loading theme configuration from ""
Aug 25 02:59:03 test-workstation sddm[29461]: Display server starting...
Aug 25 02:59:03 test-workstation sddm[29461]: Adding cookie to "/var/run/sddm/{0d6122a1-ac09-4d45-852e-8ef2ea1e037a}"
Aug 25 02:59:03 test-workstation sddm[29461]: Running: /usr/bin/X -nolisten tcp -auth /var/run/sddm/{0d6122a1-ac09-4d45-852e-8ef2ea1e037a} -background none -noreset -displayfd 17 -seat seat0>
Aug 25 02:59:04 test-workstation sddm[29461]: Failed to read display number from pipe
Aug 25 02:59:04 test-workstation sddm[29461]: Could not start Display server on vt 1
```

Then this one is a much more complicated issue: The X Server now is truly failing to start. You have to first inspect the real crash from the Xorg's own log: 
```bash
sudo cat /var/log/Xorg.0.log | tail -60
```

My log gives me this: 
```bash
[  1551.485] (II) LoadModule: "fb"
[  1551.485] (II) Module "fb" already built-in
[  1551.485] (II) UnloadModule: "fbdev"
[  1551.485] (II) Unloading fbdev
[  1551.485] (II) UnloadSubModule: "fbdevhw"
[  1551.485] (II) Unloading fbdevhw
[  1551.486] (II) UnloadModule: "vesa"
[  1551.486] (II) Unloading vesa
[  1551.601] (==) modeset(0): Backing store enabled
[  1551.601] (==) modeset(0): Silken mouse enabled
[  1551.602] (II) modeset(0): Initializing kms color map for depth 24, 8 bpc.
[  1551.602] (==) modeset(0): DPMS enabled
[  1551.602] (II) modeset(0): [DRI2] Setup complete
[  1551.602] (II) modeset(0): [DRI2]   DRI driver: nouveau
[  1551.602] (II) modeset(0): [DRI2]   VDPAU driver: nouveau
[  1551.602] (II) Initializing extension Generic Event Extension
[  1551.602] (II) Initializing extension SHAPE
[  1551.602] (II) Initializing extension MIT-SHM
[  1551.603] (II) Initializing extension XInputExtension
[  1551.603] (II) Initializing extension XTEST
[  1551.603] (II) Initializing extension BIG-REQUESTS
[  1551.603] (II) Initializing extension SYNC
[  1551.603] (II) Initializing extension XKEYBOARD
[  1551.603] (II) Initializing extension XC-MISC
[  1551.603] (II) Initializing extension SECURITY
[  1551.603] (II) Initializing extension XFIXES
[  1551.603] (II) Initializing extension RENDER
[  1551.603] (II) Initializing extension RANDR
[  1551.603] (II) Initializing extension COMPOSITE
[  1551.604] (II) Initializing extension DAMAGE
[  1551.604] (II) Initializing extension MIT-SCREEN-SAVER
[  1551.604] (II) Initializing extension DOUBLE-BUFFER
[  1551.604] (II) Initializing extension RECORD
[  1551.604] (II) Initializing extension DPMS
[  1551.604] (II) Initializing extension Present
[  1551.604] (II) Initializing extension DRI3
[  1551.604] (II) Initializing extension X-Resource
[  1551.604] (II) Initializing extension XVideo
[  1551.604] (II) Initializing extension XVideo-MotionCompensation
[  1551.604] (II) Initializing extension SELinux
[  1551.604] (II) SELinux: Disabled on system
[  1551.604] (II) Initializing extension GLX
[  1551.878] (EE) AIGLX error: Calling driver entry point failed
[  1551.885] (II) IGLX: Loaded and initialized swrast
[  1551.885] (II) GLX: Initialized DRISWRAST GL provider for screen 0
[  1551.885] (II) Initializing extension XFree86-VidModeExtension
[  1551.885] (II) Initializing extension XFree86-DGA
[  1551.885] (II) Initializing extension XFree86-DRI
[  1551.885] (II) Initializing extension DRI2
[  1551.885] (EE) modeset(0): Failed to create pixmap
[  1551.885] (EE) 
Fatal server error:
[  1551.885] (EE) failed to create screen resources(EE) 
[  1551.885] (EE) 
Please consult the The X.Org Foundation support 
         at http://wiki.x.org
 for help. 
[  1551.885] (EE) Please also check the log file at "/var/log/Xorg.0.log" for additional information.
[  1551.885] (EE) 
[  1551.891] (EE) Server terminated with error (1). Closing log file.
```

Look at this line: 
```bash
[  1551.602] (II) modeset(0): [DRI2]   DRI driver: nouveau
```

The modeset(0) is loaded with a nouveau backend instead of my NVIDIA Driver. Do another check: 
```bash
dpkg -l | grep -i nouveau
```

If you get this, then you have a same problem with me: 
```bash
ii  libdrm-nouveau2:amd64                         2.4.113-2~ubuntu0.22.04.1                        amd64        Userspace interface to nouveau-specific kernel DRM services -- runtime
ii  xserver-xorg-video-nouveau                    1:1.0.17-2build1                                 amd64        X.Org X server -- Nouveau display driver 
```

Sequence of commands to resolve this: 
```bash
# 1. Remove nouveau so Xorg can't fall back to it
sudo apt remove -y xserver-xorg-video-nouveau

# 2. Regenerate xorg.conf with the NVIDIA driver explicitly
sudo nvidia-xconfig --allow-empty-initial-configuration --enable-all-gpus

# 3. Check it picked "nvidia" as the driver
cat /etc/X11/xorg.conf
```

If the third command gives you the Driver "nvidia" in the Device section, then you do this: 

```bash
sudo nano /etc/X11/xorg.conf
```

Then inside the Section "Device" ... EndSection, add these two lines:
```bash
Option "ConnectedMonitor" "DFP-0"
Option "AllowEmptyInitialConfiguration" "true"
```

Save and exit, then continue: 
```bash
# 4. Restart the display manager
sudo systemctl restart sddm

# 5. Check the fresh Xorg log for success or errors, no EE means okay
sudo cat /var/log/Xorg.0.log | tail -40

# 6. Confirm a live graphical session
ps aux | grep -i xorg
loginctl list-sessions
```

Setup up SDDM autologin: 
```bash
sudo mkdir -p /etc/sddm.conf.d
sudo tee /etc/sddm.conf.d/autologin.conf <<'EOF'
[Autologin]
User=your-google-cloud-user-name
Session=plasma.desktop
EOF
sudo systemctl restart sddm
```

Check again: 
```bash
loginctl list-sessions
ps aux | grep -i plasma
```

You should now see a session owned by you on seat0, and multiple plasma-related processes (like plasmashell, kwin_x11) running. If not, ask Claude LOL. 

Restart sunshine and proceed again: 
```bash
systemctl --user restart sunshine
journalctl --user -u sunshine -f
```

If this doc has any issues, you are welcome to open an issue/PR.  