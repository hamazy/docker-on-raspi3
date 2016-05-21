Docker on Raspberry Pi 3
========================

The original document I referred to is "[Run Docker on a Raspberry Pi 3 with onboard WiFi](http://blog.hypriot.com/post/run-docker-rpi3-with-wifi/)".

## Create a micro SD card

Download Raspbian Jessie LITE from [the official page](https://www.raspberrypi.org/downloads/raspbian/).

~~~
$ curl -O http://vx2-downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2016-05-13/2016-05-10-raspbian-jessie-lite.zip
~~~

Calculate the SHA1 hash to confirm the image is valid.

~~~
$ shasum 2016-05-10-raspbian-jessie-lite.zip
333bfc855e8944ecb1142337ead8c928dc6c9d95  2016-05-10-raspbian-jessie-lite.zip
~~~

Insert a micro SD card. Unmount it with Disk Utility if it's mounted. But don't eject it yet.

Find the device path like `/dev/disk2` for the SD card.

~~~
$ diskutil list
~~~

Write the image to the SD card.

~~~
$ unzip -p 2016-05-10-raspbian-jessie-lite.zip | sudo dd of=/dev/disk2 bs=1m
~~~

## Booting and WiFi and SSH configuration

Insert the SD card to your raspi 3 and power it on to boot it.

Since serial console doesn't work properly (See
[Serial console broken on RPi 3](https://github.com/RPi-Distro/repo/issues/22)),
I had to connect my raspi3 with my display and keyboard.

Login with the initial user `pi` and password `raspberry`.

Edit `/etc/wpa_supplicant/wpa_supplicant.conf` to add the following.
Replace with the actual SSID and pre-shared key.
~~~
$ sudo ed /etc/wpa_supplicant/wpa_supplicant.conf <<EOF
network={
  ssid="your-ssid"
  psk="your-psk"
}
EOF
~~~

I added my ssh public key to `authorized_keys`.

~~~
$ mkdir .ssh
$ chmod 700 .ssh
$ ed <<EOF
f .ssh/authorized_keys
a
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDRBdTxEPqF4NTBocTcVipsrXqVXPpM8ZfUxK2/SlYh2hLefPAnfz4eFw72dRUG8JnXpvQu1de9DCKL9s51XbZZ0B1Clbvn8Rx9f4zhK+SReFRk4E1jLPnZKwrfK0/wb723QidLDYpNAtE7wWKY/JyMcfhteLLf+MpmbM/LJgPsrPsSOIIw3xOqWhOLQWe6wPHgGLKLisR+yGMSDi6l3EzpdKRxRQJMot3u0rY4yLlf5YEIgXPWwZVlAHVV7hFlQCZwsBcs2EPKZ/bEodvAdzGKsiBWvuJOGpbOKewYkuW3ws5CKXIXj6eRrTjM2Uey9ab/o2LLHpiDrwytPrk/ho69
.
w
q
EOF
$ chmod 600 .ssh/authorized_keys
~~~

After restarting, wifi should be connected.

~~~
$ sudo shutdown -r now
~~~

At this point, I can log in to the machine via ssh. Thanks to avahi-daemon, the machine should be reachable if your ssh client is on the same network. That means, you can ssh with the following command.

~~~
$ ssh pi@raspberrypi.local
~~~

## Updating and some initial configuration

Once it's connected to the Internet, you can update the system:

~~~
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade
~~~

Enable `overlay` module as docker requires it.

~~~
$ sudo ed /etc/modules <<EOF
a
overlay
.
w
q
EOF
~~~

With `raspi-config` command, I configured the following:

* 1 Expand Filesystem
* 5 Internationalisation Options
  * I2 Change Timezone -> Asia/Tokyo
  * I4 Change Wi-fi Country -> JP
* 9 Advanced Options
  * A7 Serial -> Yes

~~~
$ sudo raspi-config
~~~

When you exit from `raspi-config`, the system is restarted.

## Configure Docker

After restart, I used `ansible` for the remaining configuration.

~~~
$ ansible-playbook -u pi -i inventry.txt docker-on-raspi3.yml
~~~

At this moment, `docker` is ready.

~~~
$ sudo docker version
Client:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.4.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 22:23:48 2016
 OS/Arch:      linux/arm

Server:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.4.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 22:23:48 2016
 OS/Arch:      linux/arm
~~~

~~~
$ sudo docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.10.3
Storage Driver: overlay
 Backing Filesystem: extfs
Execution Driver: native-0.2
Logging Driver: json-file
Plugins:
 Volume: local
 Network: host bridge null
Kernel Version: 4.4.9-v7+
Operating System: Raspbian GNU/Linux 8 (jessie)
OSType: linux
Architecture: armv7l
CPUs: 4
Total Memory: 925.5 MiB
Name: raspberrypi
ID: XX6L:OK5X:JUEE:JRFN:ATVN:R2DJ:ERWS:D552:THLC:DU4L:6W6W:4GOF
Debug mode (server): true
 File Descriptors: 11
 Goroutines: 20
 System Time: 2016-05-21T14:51:54.608749253+09:00
 EventsListeners: 0
 Init SHA1: 0db326fc09273474242804e87e11e1d9930fb95b
 Init Path: /usr/lib/docker/dockerinit
 Docker Root Dir: /var/lib/docker
WARNING: No swap limit support
WARNING: No cpu cfs quota support
WARNING: No cpu cfs period support
WARNING: No cpuset support
~~~

Start a new docker container:

~~~
$ docker run -d -p 80:80 hypriot/rpi-busybox-httpd
~~~

List docker containers:

~~~
$ docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                NAMES
3526f4d5a182        hypriot/rpi-busybox-httpd   "/bin/busybox httpd -"   11 minutes ago      Up 11 minutes       0.0.0.0:80->80/tcp   sleepy_swirles
~~~
