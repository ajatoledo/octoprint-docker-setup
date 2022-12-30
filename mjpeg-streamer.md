# `mjpg-streamer` Outside of Octoprint

⚠️ **WARNING**: `mjpg-streamer` should not be used on untrusted networks! By default, anyone with access to the network that `mjpg-streamer` is running on will be able to access it

## Running `mjpg-streamer` Outside of Octoprint

Users may choose to run `mjpg-streamer` camera stream outside of Octoprint, but working with `mjpg-streamer` isn't always *easy*. 

The steps below outline how to configure:

1. a Bash script to launch camera streams 

2. a `systemd` process to own and launch the `mjpg-streamer` script

## Solution

- Configure a `systemd` process that starts at boot to activate `mjpg-streamer` camera streams
    
    + **Note**: `mjpg-streamer` is *challenging* [to install](https://github.com/jacksonliam/mjpg-streamer), so to simplify the process, the [snap version](https://snapcraft.io/mjpg-streamer) is used. Therefore, [snapd](https://snapcraft.io/docs/installing-snapd) is required

## Install `mjpg-streamer`

The steps below install `mjpg-streamer` using the ubuntu snap package. There are alternative approaches for installing `mjpg-streamer` [1](https://github.com/john-clark/mjpg-streamer-setup), [2](https://www.acmesystems.it/video_streaming), but I have not tested/verified whether they work, so *YMMV*.

### Installing via Snap

Installation is easy, run the two steps below.

- Snap Install: `sudo snap install mjpg-streamer`

- Update Snap Permissions for Camera: `sudo snap connect mjpg-streamer:camera`

    + [Snap Source Page](https://snapcraft.io/mjpg-streamer)
    + [Source Page](https://github.com/ogra1/mjpg-streamer)

---

## Start Camera Stream(s)

The examples below outline open a camera stream using `mjpg-streamer` 

*I haven't found detailed documentation for `mjpg-streamer`, but this [page](http://skillfulness.blogspot.com/2010/03/mjpg-streamer-documentation.html) was helpful*

### Open A Stream For A Single Camera

- command: `sudo mjpg-streamer -i "input_uvc.so -d /dev/prusa_camera_5 -r 1280x720" -o "output_http.so -p 8081"`

    + Camera: `/dev/prusa_camera_5`
    + Resolution: 1280x720
    + Web Server: output_http.so
    + Port: 8081

To view output, visit the following links

- Stream URL: http://localhost:8081/?action=snapshot
- Snapshot URL: http://localhost:8081/?action=stream

**Note**: `localhost` will need to be replaced with the host's local ip address so that they can be accessed by other users on the network. This is necessary if hosting Octoprint using Docker

### Open Multiple Camera Streams
    
- command: `mjpg-streamer -i "input_uvc.so -d /dev/prusa_camera_5 -r 1280x720" -i "input_uvc.so -d /dev/prusa_camera_6 -r 1280x720" -o "output_http.so -p 8081"`

    + Camera_0: `/dev/prusa_camera_5`
    + Camera_1: `/dev/prusa_camera_6`
    + Resolution: 1280x720
    + Web Server: output_http.so
    + Port: 8081

*For multiple cameras, the feeds are output in the order they were input and starting at zero*

- Camera_0 Stream: http://localhost:8081/?action=stream_0
- Camera_1 Stream: http://localhost:8081/?action=stream_1
- Snapshot_0 URL: http://localhost:8081/?action=stream_0
- Snapshot_1 URL: http://localhost:8081/?action=stream_1

**Note**: `localhost` will need to be replaced with the host's local ip address so that they can be accessed by other users on the network. This is necessary if hosting Octoprint using Docker

---

## Bash Script to Activate Camera Stream

Creating a simple Bash script to activate the camera streams via `mjpg-streamer` makes activating the stream easy.

### Example Camera Stream Script

Below is an example Bash script

```
#!/usr/bin/env bash

# since there are multiple cameras, there will be multiple inputs for mjpg-streamer
# the feeds are then output in the order they were input and start at zero

# so, in the example below:

mjpg-streamer -i "input_uvc.so -d /dev/prusa_camera_5 -r 1280x720" -i "input_uvc.so -d /dev/prusa_camera_6 -r 1280x720" -o "output_http.so -p 8081"

# argument listing for clarity, don't need to update and can delete

# Camera_0: `/dev/prusa_camera_5`
# Camera_1: `/dev/prusa_camera_6`
# Resolution: 1280x720
# Web Server: output_http.so
# Port: 8081

# camera_5 stream: http://localhost:8081/?action=stream_0
# camera_6 stream: http://localhost:8081/?action=stream_1

# reference:
# https://github.com/jacksonliam/mjpg-streamer/issues/186
# https://github.com/jacksonliam/mjpg-streamer/blob/master/mjpg-streamer-experimental/plugins/output_http/README.md
```

---

### Create Script to Activate Camera Stream

1. Enter `sudo nano /usr/local/bin/camera-stream`

2. Paste the materials from the example script 👆; 

    1. *Be sure to update to match your own system.*

3. After adding the information for the script, enter `CTRL` + `O` to save the file, and then hit `ENTER`

4. To close the file, enter `CTRL` + `X`

5. Make the script executable by entering: `sudo chmod +x /usr/local/bin/camera-stream`

## `systemd` Service

Create a `systemd` process file used to create the camera feed service, which can be launched at boot

**Note**: `mjpg-streamer` is a snap package, users can use a [snap daemon](https://snapcraft.io/docs/services-and-daemons), but I prefer systemd. The steps below use systemd

### Example `systemd` Service

```
[Unit]
Description=mjpg-streamer camera feeds

[Service]
Restart=always
RestartSec=1
ExecStart=/usr/local/bin/camera-stream
ExecStop=killall mjpg_streamer

[Install]
WantedBy=multi-user.target
```

**Note**: if you save your camera script using a name different than `camera-stream` or to a different location that `/usr/local/bin`, update the `ExecStart` line in the service file above accordingly 

---

### Create `systemd` Service

1. Enter `sudo nano /etc/systemd/system/printer-stream.service`

2. Paste the materials from the example process 👆

    1. *Be sure to update to match your system.*

3. After adding the information for the script, enter `CTRL` + `O` to save the file, and then hit `ENTER`

4. To close the file, enter `CTRL` + `X`

5. To start the process, enter `sudo systemctl start printer-stream.service`
    
    1. To check whether the process started successfully, enter `sudo systemctl status printer-stream.service`
        
        1. If the status indicates that the process failed, check the `journalctl` for any errors. Depending on how long your system has been up, there may be a lot of log information to review. Running `journalctl -b | grep printer-stream` will limit the log information to the current boot, which should make checking output for troubleshooting easier
        
        2. For additional `journalctl` information consider reviewing these sites, [1](https://www.digitalocean.com/community/tutorials/how-to-use-journalctl-to-view-and-manipulate-systemd-logs) and [2](https://linuxhint.com/journalctl-tail-and-cheatsheet/)
    
6. The `printer-stream.service` process will not persist across reboots, but setting it to start at boot is easy; enter `sudo systemctl enable printer-stream.service`

## FAQ

**I don't see all of my cameras, what's up?**

While the steps above don't account for all specific system setups, presuming you tested the `mjpg-streamer` script above and it started, the issue may be with the [PCIe bandwidth available on your system](https://obsproject.com/forum/threads/2-usb-cameras-work-but-not-3.124695/post-465302). 

For example, I initially plugged four cameras into a single, powered USB 3.0 hub and could not get all of my cameras to appear. However, I could see the streams after I moved two cameras to different USB ports. 
