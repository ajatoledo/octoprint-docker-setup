
# Persist Printer and Camera Connections in Octoprint Across Reboots

## Challenges in Managing Printer and Camera Connections Across Reboots

**Background**

I use [Octoprint](https://octoprint.org/) as my webserver to manage multiple 3D printers with cameras on an x86 machine. Rather than create separate Octoprint instances on the host, I run *multiple* Octoprint [docker containers](https://hub.docker.com/r/octoprint/octoprint)

For reference, system specs for my host are provided below:

- **System Specs**

    + **OS**: Ubuntu 22.04.1 LTS x86_64
    + **Kernel**: 5.15.0-56-generic
    + **CPU**: Intel i3-8145U (4) @ 3.900GHz
    + **Memory**: 3790MiB

**Problem**

TTY are randomly assigned to devices at boot, causing depending services/programs instabilities. So they could indeed fail to start because of different TTY configurations.

As a result, my camera and 3D-printer mappings are *always* wrong after a reboot.

**Solution**

- Assign immutable TTY names to USB devices by creating symbolic links of physical devices

- Configure then services/programs to point to these symbolic TTYs

---

### Verify User Permissions

For the following steps, users will need both `sudo` and `video` permissions.

- To see the groups a user has access to enter: `groups $USER`

- If a `sudo` and/or `video` are missing, follow the steps below

**Add User to Sudoers**

- To add a user to `sudo`, run the command, `usermod -aG sudo username` as root or another sudo user

    + Check out this [page](https://linuxize.com/post/how-to-add-user-to-sudoers-in-ubuntu/) for a more detailed discussion of `sudo` permissions

**Add User to Video**

- To add a user to `video`, run the command, `sudo usermod -a -G video username`

📝 Make sure you change “username” with the name of the user that you want to grant permissions to

**Verify Group Updates**

- To see if the group(s) updated successfully enter: `groups $USER`

    + If the new group(s) isn't listed, then log out and log back in
    
    + Once logged back in, enter: `groups $USER`, and the updated group(s) should be visible

### List Camera Connections

**Required Packages**:

- `v4l-utils`

- `ffmpeg` - used for advanced testing

**List Cameras**

1. If `v4l-utils` and `ffmpeg` are already installed, move to **Step 2**, otherwise install `v4l-utils` and `ffmpeg`; enter this command: `sudo apt update && sudo apt install v4l-utils ffmpeg -y`

2. List cameras, `v4l2-ctl --list-devices`

Returns output similar to what is shown below:

```
C270 HD WEBCAM (usb-0000:00:14.0-1):
    /dev/video0
    /dev/video1
    /dev/media0

C270 HD WEBCAM (usb-0000:00:14.0-3):
    /dev/video2
    /dev/video3
    /dev/media1
```

3. For additional camera actions, review this [StackExchange Post](https://askubuntu.com/questions/348838/how-to-check-available-webcams-from-the-command-line)

---

### List Printer Serial Connections

**Required Packages**:

- `python3.10-venv`

- `pyserial`

**Install `pyserial` and Identify USB Printers**

1. If `pyserial` is already installed and is available in the environment, skip to **Step 6**; otherwise proceed

2. Create a python environment, into which pyserial will be installed; below is an example

    1. If `venv` is installed, move to **Step ii**, otherwise run this command: `sudo apt install python3.10-venv`

    2. `mkdir tools`

    3. `python3 -m venv tools`
    
    4. Activate the new python environment using the following command: `source tools/bin/activate`. If successful, a `(tools)` should appear to the left of the username in the shell
        
        1. If the activation was not successful, review the path passed above to ensure it is correct; the following steps will not work unless the environment is active

4. Install `pyserial` using the following command: `pip install pyserial`

6. Run `python -m serial.tools.miniterm`

7. Review output; enter `CTRL` + `C` to exit

Returns output similar to what is shown below:

```
--- Available ports:
---  1: /dev/ttyACM0         'Original Prusa i3 MK3'
---  2: /dev/ttyACM1         'Original Prusa MINI'
---  3: /dev/ttyS0           'ttyS0'
---  4: /dev/ttyS1           'ttyS1'
--- Enter port index or full name:
```

*If there are multiple printers, there may be duplicated names. While there are ways to test, probably the easiest is to plug in one at a time and then use the steps below to create a symlink to map the USB*

- [Reference](https://pyserial.readthedocs.io/en/latest/tools.html)

---

## Create USB Symlink Mappings

The steps below will create udev rules that will create symlinks that users can reference and result in consistent device mappings 

- `/udev/video*` and `/udev/tty*` mappings can change on each reboot which means printers and cameras may not appear reliably

### `udevadm` Overview

- The `udevadm` command is a device management tool in Linux that manages all the device events and controls the udevd daemon 

- Udev rules are defined with the .rules file in `/usr/lib/udev/rules.d`; the syntax for the `udevadm` command are provided below:

```
udevadm [--debug] [--version] [--help]
udevadm info options
udevadm trigger [options]
udevadm settle [options]
udevadm control command
udevadm monitor [options]
udevadm test [options] devpath

```

- [Source](https://www.linuxfordevices.com/tutorials/linux/udevadm-command)

---

#### `udevadm info`

- Udevadm info starts with the device specified by the devpath and then
walks up the chain of parent devices 

- It prints for every device found, all possible attributes in the udev rules key format

- A rule to match will be composed of the attributes of the device
and the attributes from one single parent device

- To *walk* the attributes of a webcam located at `/dev/video0`, run the following command: `/dev/video0`: `udevadm info --name=/dev/video0 --attribute-walk`

The command should return output similar to what is shown below:

```
Udevadm info starts with the device specified by the devpath and then
walks up the chain of parent devices. It prints, for every device
found, all possible attributes in the `udev` rules key format.
A rule to match can be composed of the attributes of the device
and the attributes from one single parent device.

  looking at device '/devices/pci0000:00/0000:00:14.0/usb1/1-1/1-1:1.0/video4linux/video0':
    KERNEL=="video0"
    SUBSYSTEM=="video4linux"
    DRIVER==""
    ATTR{dev_debug}=="0"
    ATTR{index}=="0"
    ATTR{name}=="C270 HD WEBCAM"
    ATTR{power/async}=="disabled"
    ATTR{power/control}=="auto"
    ATTR{power/runtime_active_kids}=="0"
    ATTR{power/runtime_active_time}=="0"
    ATTR{power/runtime_enabled}=="disabled"
    ATTR{power/runtime_status}=="unsupported"
    ATTR{power/runtime_suspended_time}=="0"
    ATTR{power/runtime_usage}=="0"

--- truncated
```

- If an error is returned, it likely indicates that no webcam is found at the `/dev/video0`; to get a listing of all connected webcams, run the following command: `ls /dev | grep video*` to get a list of connected webcams

**Using Serial Number Attribute**

The attribute output provides a detailed listing of attributes; however, my mappings use the serial number attribute and work well, so to limit the output returned, use the following command:

- `udevadm info --name=/dev/ttyACM0 --attribute-walk | grep serial` which produces less information 

- Below is an example of the output created

```
    ATTRS{serial}=="SERIAL_XXXXXXXXXX"
    ATTRS{serial}=="SERIAL_XXXXXXXXXX"
```

- Run this command on `/dev/video0`: `udevadm info --name=/dev/video0 --attribute-walk`

With the output, symlink rules can be created using the form presented below in the example `udev` rules

- [Source](https://gist.github.com/mspacek/96673b195fa0b16f9e20)

#### Example `udev` Rules

The rules below were created using attributes from 👆 for both printers and cameras

```
# printer 5 - prusa mk3s - mosquito
KERNEL=="ttyACM*", ATTRS{serial}=="SERIAL_XXXXXXXXXX", SYMLINK+="prusa_5"

# camera 5
KERNEL=="video*", ATTR{index}=="0", ATTRS{serial}=="SERIAL_XXXXXXXXXX", SYMLINK+="prusa_camera_5"

# printer 6 - prusa mini
KERNEL=="ttyACM*", ATTRS{serial}=="SERIAL_XXXXXXXXXX", SYMLINK+="prusa_6"

# camera 6
KERNEL=="video*", ATTR{index}=="0", ATTRS{serial}=="SERIAL_XXXXXXXXXX", SYMLINK+="prusa_camera_6"
```

**Notes**:

- The serial number information provided above suppressed my actual serial number information, in favor of a generic string, `SERIAL_XXXXXXXXXX`; update your `udev` rules to match your serial output

- The value of `SYMLINK+=""` determines the symlink values that will be used

    + My examples above use a `prusa_` pre-fix for printers and a `prusa_camera_` pre-fix for cameras; update `SYMLINK+=""` to match your needs

---

### Add Rules to `udev`

The rules 👆 were saved in `/etc/udev/rules.d/99-usb-serial.rules`

Use your preferred text editor, the example below uses [nano](https://www.nano-editor.org/)

1. Enter `sudo nano /etc/udev/rules.d/99-usb-serial.rules`

2. Paste the rules created 👆 in the editor window

3. After adding rules, enter `CTRL` + `O` to save the file, and then hit `ENTER`

4. To close the file, enter `CTRL` + `X`

---

### Trigger Rules

To trigger the new udev rules, run the following: `sudo udevadm control --reload-rules && sudo udevadm trigger`

- To verify that the new device symlinks are present, run `ls -l /dev/prusa*`, the symlink names should appear in the list


```
lrwxrwxrwx  1 root root             7 Dec 28 19:31 prusa_5 -> ttyACM0
lrwxrwxrwx  1 root root             7 Dec 28 19:31 prusa_6 -> ttyACM1
lrwxrwxrwx  1 root root             6 Dec 28 19:31 prusa_camera_5 -> video2
lrwxrwxrwx  1 root root             6 Dec 28 19:31 prusa_camera_6 -> video0
```

**Note**: the ls command above returned items using my custom symlink names which each included `prusa` as part of the process. Therefore, if different symlink names are used, update the ls command accordingly
