# Run `mjpg-streamer` Outside of Octoprint

From the [mjpg-streamer repository](https://github.com/jacksonliam/mjpg-streamer#mjpg-streamer)

>`mjpg-streamer` is a command line application that copies JPEG frames from one or more input plugins to multiple output plugins. It can be used to stream JPEG files over an IP-based network from a webcam to various types of viewers such as Chrome, Firefox, Cambozola, VLC, mplayer, and other software capable of receiving MJPG streams.
>
>It was originally written for embedded devices with very limited resources in terms of RAM and CPU. Its predecessor "uvc_streamer" was created because Linux-UVC compatible cameras directly produce JPEG-data, allowing fast and perfomant M-JPEG streams even from an embedded device running OpenWRT. The input module "input_uvc.so" captures such JPG frames from a connected webcam. `mjpg-streamer` now supports a variety of different input devices.

***Guide Notes***

- This guide will not address all camera configuration options available in `mjpg-streamer`; users needing additional information should review the [project's repository documentation](https://github.com/jacksonliam/mjpg-streamer/tree/master/mjpg-streamer-experimental)

- The example camera configurations below use two camera devices--`/dev/video0` and `/dev/video2`--which may differ from user systems. Users who want to get device names can follow the steps outlined [here](https://github.com/ajatoledo/octoprint-docker-setup/blob/main/udev-mapping.md#list-camera-connections)
    
    + Users interested in creating symlink mappings for devices should review [Persist Printer and Camera Connections in Octoprint Across Reboots](https://github.com/ajatoledo/octoprint-docker-setup/blob/main/udev-mapping.md#persist-printer-and-camera-connections-in-octoprint-across-reboots)

---

⚠️ **WARNING**: `mjpg-streamer` should not be used on untrusted networks! By default, anyone with access to the network that `mjpg-streamer` is running on will be able to access it

---

## Deploying `mjpg-streamer`

Users can install `mjpg-streamer` using the directions provided on the [project's repository](https://github.com/jacksonliam/mjpg-streamer#building--installation) to build and install `mjpg-streamer` to their system. However, the build and install process may only sometimes fit user preferences. As an alternative, two other installation approaches are outlined below and include:

### Snap Overview

[**Snap Description**:](https://en.wikipedia.org/wiki/Snap_(software)) Snap is a software packaging and deployment system developed by Canonical for operating systems that use the Linux kernel and the systemd init system. The packages called snaps, and the tool for using them, snapd, work across a range of Linux distributions and allow upstream software developers to distribute their applications directly to users. Snaps are self-contained applications running in a sandbox with mediated access to the host system. 
    
- [Install `mjpg-streamer` using snapd](#snap-install)

### Docker Overview

[**Docker Description**:](https://docs.docker.com/get-started/overview/) Docker is an open platform for developing, shipping, and running applications. Docker enables you to separate your applications from your infrastructure so you can deliver software quickly. With Docker, you can manage your infrastructure in the same ways you manage your applications. By taking advantage of Docker’s methodologies for shipping, testing, and deploying code quickly, you can significantly reduce the delay between writing code and running it in production.

- [Install `mjpg-streamer` using Docker](#docker-install)

## Install `mjpg-streamer` via Snap Package

### Snap Install

The steps below outline how to install `mjpg-streamer` using the snap package

**Note**: `snapd` is required; if it is not available on the system, review the [installation steps](https://snapcraft.io/docs/installing-snapd) provided on the [snapcraft site](https://snapcraft.io/)

### `mjpg-streamer` Install

Installation is easy; run the two commands below

- Snap Install: `sudo snap install mjpg-streamer`

- Update Snap Permissions for Camera: `sudo snap connect mjpg-streamer:camera`

    + [Snap Source Page](https://snapcraft.io/mjpg-streamer)
    + [Source Page](https://github.com/ogra1/mjpg-streamer)

### Start Camera Stream(s)

The examples below outline how to open both a single camera stream and multiple camera streams using `mjpg-streamer` 

#### Open A Stream For A Single Camera

Run the following command to open a single camera stream: `sudo mjpg-streamer -i "input_uvc.so -d /dev/video0 -r 1280x720" -o "output_http.so -p 8081"`

    + Camera: `/dev/video0`
    + Resolution: 1280x720
    + Web Server: output_http.so
    + Port: 8081

To view output, visit the following links

- Stream URL: http://localhost:8081/?action=snapshot
- Snapshot URL: http://localhost:8081/?action=stream

**Note**: `localhost` will need to be replaced with the host's local IP address so that other users on the network can access them

#### Open Multiple Camera Streams
    
Run the following command to open two camera streams: `mjpg-streamer -i "input_uvc.so -d /dev/video0 -r 1280x720" -i "input_uvc.so -d /dev/video2 -r 1280x720" -o "output_http.so -p 8081"`

    + Camera_0: `/dev/video0`
    + Camera_1: `/dev/video2`
    + Resolution: 1280x720
    + Web Server: output_http.so
    + Port: 8081

*For multiple cameras, the feeds are output in the order they were input and starting at zero*

- Camera_0 Stream: http://localhost:8081/?action=stream_0 -> device = `/dev/video0`
- Camera_1 Stream: http://localhost:8081/?action=stream_1 -> device = `/dev/video2`

- Snapshot_0 URL: http://localhost:8081/?action=stream_0 -> device = `/dev/video0`
- Snapshot_1 URL: http://localhost:8081/?action=stream_1 -> device = `/dev/video2`

**Note**: `localhost` will need to be replaced with the hosts' local IP addresses so that other users on the network can access them

### Start Camera Streams on Boot

Users can start camera stream(s) at boot by creating a [`systemd`](### `systemd` Service) or [`snapd`](### `snapd` Service) service; steps for each are outlined below. To avoid device conflicts, users should only use one service type

#### `systemd` Service

##### Step 1. Create Script to Activate Camera Stream(s)

1. Enter `sudo nano /usr/local/bin/camera-stream`

2. Paste the example script provided below into the editor

    ```
    #!/usr/bin/env bash
    
    # since there are multiple cameras, there will be multiple inputs for mjpg-streamer
    # the feeds are then output in the order they were input and start at zero
    
    # so, in the example below:
    
    mjpg-streamer -i "input_uvc.so -d /dev/video0 -r 1280x720" -i "input_uvc.so -d /dev/video2 -r 1280x720" -o "output_http.so -p 8081"
    
    # argument listing for clarity, don't need to update and can delete
    
    # Camera_0: `/dev/video0`
    # Camera_1: `/dev/video2`
    # Resolution: 1280x720
    # Web Server: output_http.so
    # Port: 8081
    
    # Camera stream URLs
    # camera_5 stream: http://localhost:8081/?action=stream_0
    # camera_6 stream: http://localhost:8081/?action=stream_1
    ```

    + *Update the script to match your system*

3. After adding the script, enter `CTRL` + `O` to save the file, and then hit `ENTER`

4. To close the file, enter `CTRL` + `X`

5. Make the script executable by entering: `sudo chmod +x /usr/local/bin/camera-stream`

##### Step 2. Create `systemd` File to Start Camera Stream(s) at Boot

1. Enter `sudo nano /etc/systemd/system/printer-stream.service`

2. Paste the example script provided below into the editor

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

    + *Update to match your camera configuration*

3. After adding the information for the process file, enter `CTRL` + `O` to save the file, and then hit `ENTER`

4. To close the file, enter `CTRL` + `X`

**Note**: if you save your camera script using a name different than `camera-stream` or to a different location that `/usr/local/bin`, update the `ExecStart` line in the service file above accordingly 

##### Step 3. Start the `systemd` Service

1. To start the process, enter `sudo systemctl start printer-stream.service`
    
    1. To check whether the process started successfully, enter `sudo systemctl status printer-stream.service`
        
        1. If the status indicates that the process failed, check the `journalctl` for any errors. Depending on how long your system has been up, there may be a lot of log information to review. Running `journalctl -b | grep printer-stream` will limit the log information to the current boot, which should make checking output for troubleshooting easier
        
        2. For additional `journalctl` information, consider reviewing these sites, [1](https://www.digitalocean.com/community/tutorials/how-to-use-journalctl-to-view-and-manipulate-systemd-logs) and [2](https://linuxhint.com/journalctl-tail-and-cheatsheet/)
    
2. The `printer-stream.service` process will not persist across reboots, but setting it to start at boot is easy; enter `sudo systemctl enable printer-stream.service`

#### `snapd` Service

`snapd` is the background service that manages and maintains system snaps

##### Create and Start `snapd` Service

1. Enter: `sudo nano /var/snap/mjpg-streamer/current/config`

2. Paste the example script provided below into the editor

    ```
    INPUTOPTS="-i "input_uvc.so -d /dev/video0 -r 1280x720" -i "input_uvc.so -d /dev/video2 -r 1280x720" -o "output_http.so"
    PORT="-p 8081"
    DAEMON="true"
    ```

    + *Update to match your camera configuration*

3. After adding the information for the process file, enter `CTRL` + `O` to save the file, and then hit `ENTER`

4. To close the file, enter `CTRL` + `X`

5. Activate the process by entering: `sudo snap restart mjpg-streamer`

## Launch `mjpg-streamer` Using Docker

### Docker Install

Docker installation requires both Docker Engine and the Docker-Compose-Plugin. Visit the pages below and follow the provided directions for your system

- **To Install Docker Engine**: Visit the Docker [Site](https://docs.docker.com/engine/install/) and select the system type that matches your specific system

- **To Install Docker-Compose-Plugin**: Visit the Docker Compose [Site](https://docs.docker.com/compose/install/linux/) and select the system type that matches your specific system

After installation, follow the steps below to launch `mjpg-streamer` containers

#### Create a Docker Container

The `mjpg-streamer` docker image is built using [ajatoledo/mjpg-streamer](https://github.com/ajatoledo/mjpg-streamer) and is available as a pre-built image [ajatoledo/mjpg-streamer](https://hub.docker.com/repository/registry-1.docker.io/ajatoledo/mjpg-streamer/general)

**Container Notes**

- The order of camera inputs for the docker image is: (1) stream output string, then (2) stream input string

- Each stream requires a separate container

**Deploy a Camera Stream**

`docker compose` is the recommended approach to deploying `mjpg-streamer`; however, a `docker run` example is also provided

- The following commands will open a single camera stream for `dev/video0` on port 8080 with a resolution of 1280x720

##### Launch a Container Using `docker run`

**Note**: Ensure each argument string below is present and enclosed with quotes. Additionally, don't change the command input order from output, input

```
docker run \
    -it \
    --device /dev/video0:/dev/video0 \
    -p 8080:8080 \
    ajatoledo/mjpg-streamer:latest \
    "output_http.so -w ./www" \
    "input_uvc.so"
```

##### Launch a Container Using `docker compose`

**Create `docker compose` configuration**

```
services:
  mjpg-streamer:
    restart: always
    # update to tag used to create image
    container_name: mjpg-streamer
    image: ajatoledo/mjpg-streamer
    devices:
      - /dev/video0:/dev/video0
    ports:
      - 8080:8080

# docker-start.sh default output and input arguments
# ensure each argument string is present and input with quotes
# don't change the command input order

    command:
      - "output_http.so -w ./www"
      - "input_uvc.so"

      # example command updates
      
      # - "output_http.so -w ./www -c admin:admin"
      # - "input_uvc.so -r 1920x1080"
```

**Launch the container using `docker compose`**

***Note***: the docker compose example uses v2 of `docker/compose`; users still on v1 should update `docker compose` -> `docker-compose`

```
docker compose -f docker-compose.yml up
```

---

#### Build `mjpg-streamer` Image Locally

Users wanting to build the image locally can use the command below to download the `mjpg-streamer` repository and build it locally

```
git clone "https://github.com/ajatoledo/mjpg-streamer.git" && cd mjpg-streamer/mjpg-streamer-experimental && docker build -t mjpg-streamer .
```

**Notes**: 

1. The example requires `git`; install it if it's missing 

2. The example builds docker image named `mjpg-streamer`

## FAQ

**I don't see all of my cameras; what's up?**

While the steps above don't account for all specific system setups, presuming you tested the `mjpg-streamer` script above and it started, the issue may be with the [PCIe bandwidth available on your system](https://obsproject.com/forum/threads/2-usb-cameras-work-but-not-3.124695/post-465302). 

For example, I initially plugged four cameras into a single, powered USB 3.0 hub and could not get all of my cameras to appear. However, I could see the streams after I moved two cameras to different USB ports. 
