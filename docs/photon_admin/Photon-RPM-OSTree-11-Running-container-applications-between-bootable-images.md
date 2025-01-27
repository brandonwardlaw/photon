# Running container applications between bootable images

In this chapter, we want to test a docker application and make sure that all the settings and downloads done in one bootable filetree are going to be saved into writable folders and be available in the other image, in other words after reboot from the other image, everything is available exactly the same way.   
We are going to do this twice: first, to verify an existing bootable image installed in parallel and then create a new one.

## Downloading a docker container appliance

Photon OS comes with docker package installed and configured, but we expect that the docker daemon is inactive (not started). Configuration file /usr/lib/systemd/system/docker.service is read-only (remember /usr is bound as read-only). 
```
root@sample-host-def [ ~ ]# systemctl status docker
* docker.service - Docker Daemon
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled)
   Active: inactive (dead)

root@sample-host-def [ ~ ]# cat /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

Now let's enable docker daemon to start at boot time - this will create a symbolic link into writable folder /etc/systemd/system/multi-user.target.wants to its systemd configuration, as with all other systemd controlled services. 

```
root@sample-host-def [ ~ ]# systemctl enable docker
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service -> /lib/systemd/system/docker.service.

root@sample-host-def [ ~ ]# ls -l /etc/systemd/system/multi-user.target.wants
total 0
lrwxrwxrwx 1 root root 34 Sep 10 10:48 docker.service -> /lib/systemd/system/docker.service
lrwxrwxrwx 1 root root 36 Sep  4 04:59 iptables.service -> /lib/systemd/system/iptables.service
lrwxrwxrwx 1 root root 35 Sep  4 04:59 machines.target -> /lib/systemd/system/machines.target
lrwxrwxrwx 1 root root 36 Sep  4 04:59 remote-fs.target -> /lib/systemd/system/remote-fs.target
lrwxrwxrwx 1 root root 39 Sep  4 04:59 sshd-keygen.service -> /lib/systemd/system/sshd-keygen.service
lrwxrwxrwx 1 root root 32 Sep  4 04:59 sshd.service -> /lib/systemd/system/sshd.service
lrwxrwxrwx 1 root root 44 Sep  4 04:59 systemd-networkd.service -> /lib/systemd/system/systemd-networkd.service
lrwxrwxrwx 1 root root 44 Sep  4 04:59 systemd-resolved.service -> /lib/systemd/system/systemd-resolved.service
```
To verify that the symbolic link points to a file in a read-only directory, try to make a change in this file using vim and save. you'll get an error: "/usr/lib/systemd/system/docker.service" E166: Can't open linked file for writing".  

Finally, let's start the daemon, check again that is active.

```
root@sample-host-def [ ~ ]# systemctl start docker

root@sample-host-def [ ~ ]# systemctl status -l docker
* docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-09-10 10:54:32 UTC; 14s ago
     Docs: https://docs.docker.com
 Main PID: 2553 (dockerd)
    Tasks: 35 (limit: 4711)
   Memory: 148.2M
   CGroup: /system.slice/docker.service
           |-2553 /usr/bin/dockerd
           `-2566 docker-containerd --config /var/run/docker/containerd/containerd.toml

Sep 10 10:54:31 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:31.421759662Z" level=info msg="pickfirstBalancer: HandleSubConnStateChange: 0xc420312f90, CONNECTING" module=grpc
Sep 10 10:54:31 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:31.421935355Z" level=info msg="pickfirstBalancer: HandleSubConnStateChange: 0xc420312f90, READY" module=grpc
Sep 10 10:54:31 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:31.421980614Z" level=info msg="Loading containers: start."
Sep 10 10:54:31 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:31.886520281Z" level=info msg="Default bridge
(docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used to set a preferred IP address"
Sep 10 10:54:32 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:32.027763113Z" level=info msg="Loading containers: done."
Sep 10 10:54:32 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:32.468277184Z" level=info msg="Docker daemon"
commit=6d37f41 graphdriver(s)=overlay2 version=18.06.2-ce
Sep 10 10:54:32 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:32.468441587Z" level=info msg="Daemon has completed initialization"
Sep 10 10:54:32 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:32.684925824Z" level=warning msg="Could not register builder git source: failed to find git binary: exec: \"git\": executable file not found in $PATH"
Sep 10 10:54:32 photon-76718dd2fa33 dockerd[2553]: time="2019-09-10T10:54:32.691070166Z" level=info msg="API listen on /var/run/docker.sock"
Sep 10 10:54:32 photon-76718dd2fa33 systemd[1]: Started Docker Application Container Engine.
```

We'll ask docker to run Ubuntu Linux in a container. Since it's not present locally, it's going to be downloaded first from the official docker repository https://hub.docker.com/_/ubuntu/.

```
root@sample-host-def [ ~ ]# docker ps -a
CONTAINER ID        IMAGE            COMMAND      CREATED           STATUS              PORTS       NAMES

root@sample-host-def [ ~ ]# docker run -it ubuntu
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
35c102085707: Pull complete
251f5509d51d: Pull complete
8e829fe70a46: Pull complete
6001e1789921: Pull complete
Digest: sha256:d1d454df0f579c6be4d8161d227462d69e163a8ff9d20a847533989cf0c94d90
Status: Downloaded newer image for ubuntu:latest
```

When downloading is complete, it comes to Ubuntu root prompt with assigned host name 7029a64e7aa3, that is actually the Container ID. Let's verify it's indeed the expected OS.

```
root@sample-host-def [ ~ ]# docker run -it ubuntu
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
d3a1f33e8a5a: Pull complete
c22013c84729: Pull complete
d74508fb6632: Pull complete
91e54dfb1179: Already exists
library/ubuntu:latest: The image you are pulling has been verified. Important: image verification is a tech preview feature and should not be relied on to provide security.
Digest: sha256:fde8a8814702c18bb1f39b3bd91a2f82a8e428b1b4e39d1963c5d14418da8fba
Status: Downloaded newer image for ubuntu:latest

root@7029a64e7aa3:/# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.3 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.3 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
root@7029a64e7aa3:/#
```
Now let's write a file into Ubuntu home directory

```
echo "Ubuntu file" >> /home/myfile
root@7029a64e7aa3:/home# cat /home/myfile
Ubuntu file
```

We'll exit back to the Photon prompt and if it's stopped, we will re-start it.

```
root@7029a64e7aa3:/# exit
exit

root@sample-host-def [ ~ ]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                       PORTS               NAMES
7029a64e7aa3        ubuntu              "/bin/bash"         6 minutes ago       Exited (0) 11 seconds ago                        gifted_dijkstra

root@photon-host-cus1 [ ~ ]# docker start  7029a64e7aa3
7029a64e7aa3

root@photon-host-cus1 [ ~ ]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                       PORTS               NAMES
7029a64e7aa3        ubuntu              "/bin/bash"         7 minutes ago       Up 21 seconds                                    gifted_dijkstra
```

## Rebooting into an existing image

Now let's reboot the machine and select the other image. First, we'll verify that the docker daemon is automaically started.

```
root@photon-host-cus1 [ ~ ]# systemctl status docker
* docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2019-09-10 10:54:32 UTC; 13min ago
     Docs: https://docs.docker.com
 Main PID: 2553 (dockerd)
    Tasks: 55 (limit: 4711)
   Memory: 261.3M
   CGroup: /system.slice/docker.service
           |-2553 /usr/bin/dockerd
   ...
```

Next, is the Ubuntu OS container still there?

```
root@photon-host-cus1 [ ~ ]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                       PORTS               NAMES
7029a64e7aa3        ubuntu              "/bin/bash"         9 minutes ago       Up 2 minutes                                     gifted_dijkstra
```

It is, so let's start it, attach and verify that our file is persisted, then add another line to it and save, exit.

```
root@photon-host-cus1 [ ~ ]# docker start -i  7029a64e7aa3
root@7029a64e7aa3:/# cat /home/myfile
Ubuntu file
root@7029a64e7aa3:/# echo "booted into existing image" >> /home/myfile
root@7029a64e7aa3:/# exit
exit
```

## Reboot into a newly created image

Let's upgrade and replace the .0 image by a .3 build that contains git and also perl_YAML (because it is a dependency of git).

```
root@photon-host-cus1 [ ~ ]# rpm-ostree status
  TIMESTAMP (UTC)         VERSION               ID             OSNAME     REFSPEC
* 2015-09-04 00:36:37     1.0_tp2_minimal.2     092e21d292     photon     photon:photon/tp2/x86_64/minimal
  2015-08-20 22:27:43     1.0_tp2_minimal       2940e10c4d     photon     photon:photon/tp2/x86_64/minimal

root@photon-host-cus1 [ ~ ]# rpm-ostree upgrade
Updating from: photon:photon/tp2/x86_64/minimal

43 metadata, 209 content objects fetched; 19992 KiB transferred in 0 seconds
Copying /etc changes: 5 modified, 0 removed, 19 added
Transaction complete; bootconfig swap: yes deployment count change: 0
Freed objects: 16.2 MB
Added:
  git-2.1.2-1.ph3tp2.x86_64
  perl-YAML-1.14-1.ph3tp2.noarch
Upgrade prepared for next boot; run "systemctl reboot" to start a reboot

root@photon-host-cus1 [ ~ ]# rpm-ostree status
  TIMESTAMP (UTC)         VERSION               ID             OSNAME     REFSPEC
  2015-09-06 18:12:08     1.0_tp2_minimal.3     d16aebd803     photon     photon:photon/tp2/x86_64/minimal
* 2015-09-04 00:36:37     1.0_tp2_minimal.2     092e21d292     photon     photon:photon/tp2/x86_64/minimal
```

After reboot from 1.0_tp2_minimal.3 build, let's check that the 3-way /etc merge succeeded as expected. The docker.service slink is still there, and docker demon restarted at boot.

```
root@photon-host-cus1 [ ~ ]# ls -l /etc/systemd/system/multi-user.target.wants/docker.service
lrwxrwxrwx 1 root root 38 Sep  6 12:50 /etc/systemd/system/multi-user.target.wants/docker.service -> /usr/lib/systemd/system/docker.service

root@photon-host-cus1 [ ~ ]# systemctl status docker
* docker.service - Docker Daemon
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled)
   Active: active (running) since Sun 2015-09-06 12:56:33 UTC; 1min 27s ago
 Main PID: 292 (docker)
   CGroup: /system.slice/docker.service
           `-292 /bin/docker -d -s overlay
   ...
```

Let's revisit the Ubuntu container. Is the container still there? is myfile persisted?

```
root@photon-host-cus1 [ ~ ]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                    PORTS               NAMES
7029a64e7aa3        ubuntu              "/bin/bash"         5 days ago          Exited (0) 5 days ago                         gifted_dijkstra
55825c961f95        ubuntu              "/bin/bash"         5 days ago          Exited (127) 5 days ago                       distracted_shannon

root@photon-host-cus1 [ ~ ]# docker start 57dcac5d0490

root@57dcac5d0490:/# cat /home/myfile
Ubuntu file
booted into existing image
root@57dcac5d0490:/# echo "booted into new image" >> /home/myfile
```
