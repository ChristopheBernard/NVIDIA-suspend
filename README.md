
# Problem statement

I installed a fresh ubuntu 20.04.3 LTS from scratch beginning of Nov 2021. The distribution used the nvidia-driver version 470 from the ubuntu non-free section.

Wanting to install CUDA like on an old Ubuntu 18.04 distribution, I followed the instructions on nvidia's site. I ended up with 
- cuda-11-5 
- and the accompanying drivers version 495

The installation was successful. nvcc works and CUDA programs seem to run.

However, since then, the computer does no do suspend (to RAM). When trying to suspend, the screen chvt's to text mode and stays there. The only way out is power cycling.

I am a bit baffled. A simple upgrade of the driver leads to a downgrade in features.

I have long googled for answers on the internet and read the README file in the nvidia driver package. All the explanations seem to be OK, but for different distro/driver combinations, and there is no way to replicate them in my case.

I am surprised because this is a vanilla case, on a vanilla desktop (XPS 8920 with a GTX1070) with a vanilla linux distribution.

I looked at the suspend utilities in systemd. They seem to be deactivated 

```bash
/etc/systemd/system$ ls -l nvidia-*
lrwxrwxrwx 1 root root 9 nov.  25 10:55 nvidia-hibernate.service -> /dev/null
lrwxrwxrwx 1 root root 9 nov.  25 10:55 nvidia-resume.service -> /dev/null
lrwxrwxrwx 1 root root 9 nov.  25 10:55 nvidia-suspend.service -> /dev/null
```

Asking systemctl about that:

```bash
$ systemctl status nvidia-suspend
● nvidia-suspend.service
     Loaded: masked (Reason: Unit nvidia-suspend.service is masked.)
     Active: inactive (dead)
```

Is there a way to properly restore this suspend mechanism?

# Solution

I basically followed the instructions in [bmcbm's github page](https://gist.github.com/bmcbm/375f14eaa17f88756b4bdbbebbcfd029).

Only point was ... the files I was supposed to copy did not exist at all in the new distro. So I had to pick them from my old one.

Here is a modified script which does the job with missing files all put in one place called TMPL_PATH

Here is how it differs from the one by bmcbm:
1. I remove the symlinks pointing to `/dev/null`
2. I copy the files into `/lib/systemd/system`, because the `systemctl enable` command actually links them into `/etc/systemd/system/systemd-suspend.service.requires` and similar folders, and does it properly.

```bash
TMP_PATH=/var/tmp
TMPL_PATH=/home/user/missing-scripts

echo "options nvidia NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=${TMP_PATH}" | sudo tee /etc/modprobe.d/nvidia-power-management.conf 

sudo rm -f /etc/systemd/system/nvidia-suspend.service # if /dev/null
sudo rm -f /etc/systemd/system/nvidia-resume.service # if /dev/null
sudo rm -f /etc/systemd/system/nvidia-hibernate.service # if /dev/null

sudo install --mode 644 "${TMPL_PATH}/nvidia-suspend.service" /lib/systemd/system
sudo install --mode 644 "${TMPL_PATH}/nvidia-hibernate.service" /lib/systemd/system
sudo install --mode 644 "${TMPL_PATH}/nvidia-resume.service" /lib/systemd/system

sudo install --mode 755 "${TMPL_PATH}/nvidia" /lib/systemd/system-sleep
sudo install --mode 755 "${TMPL_PATH}/nvidia-sleep.sh" /usr/bin

sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service
```
And here are the 5 files missing to put in /home/user/missing-scripts

File `nvidia-suspend.service`:
```
[Unit]
Description=NVIDIA system suspend actions
Before=systemd-suspend.service

[Service]
Type=oneshot
ExecStart=/usr/bin/logger -t suspend -s "nvidia-suspend.service"
ExecStart=/usr/bin/nvidia-sleep.sh "suspend"

[Install]
RequiredBy=systemd-suspend.service
```
File `nvidia-resume.service`:
```
[Unit]
Description=NVIDIA system resume actions
After=systemd-suspend.service
After=systemd-hibernate.service

[Service]
Type=oneshot
ExecStart=/usr/bin/logger -t suspend -s "nvidia-resume.service"
ExecStart=/usr/bin/nvidia-sleep.sh "resume"

[Install]
RequiredBy=systemd-suspend.service
RequiredBy=systemd-hibernate.service
```
File: `nvidia-hibernate.service`:
```
[Unit]
Description=NVIDIA system hibernate actions
Before=systemd-hibernate.service

[Service]
Type=oneshot
ExecStart=/usr/bin/logger -t hibernate -s "nvidia-hibernate.service"
ExecStart=/usr/bin/nvidia-sleep.sh "hibernate"

[Install]
RequiredBy=systemd-hibernate.service
```
File: `nvidia`:
```bash
#!/bin/sh

case "$1" in
    post)
        /usr/bin/nvidia-sleep.sh "resume"
        ;;
esac
```
File: `nvidia-sleep.sh`:
```bash
#!/bin/bash

if [ ! -f /proc/driver/nvidia/suspend ]; then
    exit 0
fi

RUN_DIR="/var/run/nvidia-sleep"
XORG_VT_FILE="${RUN_DIR}"/Xorg.vt_number

PATH="/bin:/usr/bin"

case "$1" in
    suspend|hibernate)
        mkdir -p "${RUN_DIR}"
        fgconsole > "${XORG_VT_FILE}"
        chvt 63
        if [[ $? -ne 0 ]]; then
            exit $?
        fi
        echo "$1" > /proc/driver/nvidia/suspend
        exit $?
        ;;
    resume)
        echo "$1" > /proc/driver/nvidia/suspend 
        #
        # Check if Xorg was determined to be running at the time
        # of suspend, and whether its VT was recorded.  If so,
        # attempt to switch back to this VT.
        #
        if [[ -f "${XORG_VT_FILE}" ]]; then
            XORG_PID=$(cat "${XORG_VT_FILE}")
            rm "${XORG_VT_FILE}"
            chvt "${XORG_PID}"
        fi
        exit 0
        ;;
    *)
        exit 1
esac
```

I have no experience with the hibernate mode though. I suggest you proceed carefully and backup config files if you try that. No need to claim you lost 10,000 BTC in mining revenues because of me.

**Additional edit**: in my case, I also had to force the linux kernel to have a less conservative view of the BIOS ACPI capabilities. I used the following script I found [here](https://iam.tj/prototype/enhancements/Windows-acpi_osi.html). 
```bash
VERSION=$(sudo strings /sys/firmware/acpi/tables/DSDT | grep -i 'windows ' | sort | tail -1)

echo 'Linux kernel command-line parameters required: acpi_osi=! "acpi_osi='$VERSION'"'

config() { 
    sed -n '/.*linux[[:space:]].*root=\(.*\)/{s//BOOT_IMAGE=\1/ p;q;}' /boot/grub/grub.cfg; 
}

echo "Existing Command Line: ` config `"

sudo sed -i "s/^\(GRUB_CMDLINE_LINUX=.*\)\"$/\1 acpi_osi=! \\\\\"acpi_osi=$VERSION\\\\\"\"/" /etc/default/grub
sudo update-grub
echo "Modified Command Line: ` config `"
```