<!-- omit in toc -->
# Rootless Container

<!-- omit in toc -->
## Table of Contents
- [Purpose](#purpose)
- [Linux Commands](#linux-commands)
- [Golang Container](#golang-container)
- [Sources](#sources)

## Purpose

- The purpose of this mini project was to better familiarize myself with how containers work under the hood
- We will be running containers both by utilizing Linux commands, and running code in `Go` that utilizes these native Linux features

## Linux Commands

- I am running these commands in a virtual machine running Ubuntu 20.04 using [multipass](https://multipass.run/)

```bash
# Launch the VM
multipass launch \
	--name="rootless-container" \
	--cpus="2" \
	--mem="2048"

# Create a directory for the Alpine root filesystem
mkdir alpinefs
cd alpinefs

# Download an Alpine root filesystem
curl https://dl-cdn.alpinelinux.org/alpine/v3.15/releases/x86_64/alpine-minirootfs-3.15.0-x86_64.tar.gz \
	--output rootfs.tar.gz

# Extract the files
tar xzf rootfs.tar.gz && rm rootfs.tar.gz

# Chroot changes the root directory for the currently running process
sudo chroot /home/ubuntu/alpinefs /bin/sh

# List directory contents
ls

# Output
bin    etc    lib    mnt    proc   run    srv    tmp    var		dev    home   media  opt    root   sbin   sys    usr
```

- This is a good start, but right now we are running our container as the host's root
- We also cannot see currently running processes in the container

```bash
# In the container
touch file
ls -l | grep file

# Output
-rw-r--r--    1 root     root             0 Dec 30 23:23 file

# In the VM user namespace
ls -l /home/ubuntu/alpinefs | grep file

# Output
-rw-r--r--  1 root   root      0 Dec 30 15:23 file

# In the container
ps aux

# Output
PID   USER     TIME  COMMAND
```

- What want to create the illusion of running as the root user in the container, not actually run as the root user
- We also want to be able to see the processes running inside the container

```bash
# Mount the /proc folder in the Alpine filesystem
mount -t proc proc /proc

# As the VM user, run this sleep process
sleep 1500

# In the container, find the process running
ps aux | grep "sleep 1500"

# Output
9998 1000      	0:00 sleep 1500
10000 root      0:00 grep sleep 1500
```

- So now we can see all processes running on the host within the container, and can even kill them since we are running `chroot` as the host's root

```bash
# Kill the sleeping process from within the container
kill -KILL $(ps aux | grep "sleep 1500" | awk '{print $1}' | head -n 1)
```

- Still not what we want
- We need to utilizes kernal namespaces to isolate our container
- We can do this with the `unshare` command
- Exit the currently running container and start the next set of commands


```bash
# Exit the container
exit

# Mount the Alpine proc directory
sudo mount --types proc proc $(pwd)/alpinefs/proc

# Create a PID namespace and the same chroot shell as before
sudo unshare \
	--pid \
	--fork \
	--mount-proc="/home/ubuntu/alpinefs/proc" \
	chroot /home/ubuntu/alpinefs /bin/sh

# See only the processes running in this newly create PID namespace
ps aux

# Output
PID   USER     TIME  COMMAND
1 root      	0:00 /bin/sh
5 root      	0:00 ps aux
```

- In Docker containers, we can mount volumes from the host to the container
- Let's mount a readonly directory named `config` to our container

```bash
# On the host, create a config directory with some files
mkdir configs
touch configs/config1
touch configs/config2

# Create a container-configs directory in the Alpine filesystem
mkdir /home/ubuntu/alpinefs/container-configs

# Create a bind mount for the configs directory to the container
sudo mount \
	--bind \
	--read-only /home/ubuntu/configs /home/ubuntu/alpinefs/container-configs

# Run the container
sudo unshare \
	--pid \
	--fork \
	--mount-proc="/home/ubuntu/alpinefs/proc" \
	chroot /home/ubuntu/alpinefs /bin/sh

# Check permissions
ls -l container-configs/

# Output
-rw-rw-r--    1 1000     1000             0 Dec 30 23:58 config1
-rw-rw-r--    1 1000     1000             0 Dec 30 23:58 config2

# To unmount, run the command on the host
 sudo umount /home/ubuntu/alpinefs/container-configs
```

- We still need to stop running the container as root
- We can run the `unshare` as a non-root user with the `--user` flag

```bash
# From the host
unshare --user /bin/sh

# See the ID of the user in the container
id

# Output
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)

# From the container
ps -f

# Output
UID          PID    PPID  C STIME TTY          TIME CMD
nobody      8850    8849  0 13:21 pts/1    00:00:00 bash
nobody     10192    8850  0 16:17 pts/1    00:00:00 /bin/sh
nobody     10197   10192  0 16:18 pts/1    00:00:00 ps -f

# From the host
ps -t 1 -f
UID          PID    PPID  C STIME TTY          TIME CMD
ubuntu      8850    8849  0 13:21 pts/1    00:00:00 bash
ubuntu     10192    8850  0 16:17 pts/1    00:00:00 /bin/sh
```

- So even though we are running the container as `nobody`, the host user still identifies its processes as its own
- We can create the illusion of the container running as a root user by mapping the host's user id to the root id in the container

```bash
# Exit the container
exit

# Enter the container
unshare --map-root-user /bin/sh

# See the ID of the user in the container
id

# Output
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)

# From the container
ps -f

# Output
UID          PID    PPID  C STIME TTY          TIME CMD
root        8850    8849  0 13:21 pts/1    00:00:00 bash
root       10200    8850  0 16:21 pts/1    00:00:00 /bin/sh
root       10203   10200  0 16:21 pts/1    00:00:00 ps -f

# From the host
ps -t 1 -f
UID          PID    PPID  C STIME TTY          TIME CMD
ubuntu      8850    8849  0 13:21 pts/1    00:00:00 bash
ubuntu     10200    8850  0 16:21 pts/1    00:00:00 /bin/sh
```

- We now appear to be running as root in the container, but the host still sees its processes as its own
- From within this new user namespace, we can run `unshare` commands as a non-root user, or rather as an illusion of the root user

```bash
# Create new PID and UTS namespaces
unshare --pid --fork --uts --mount-proc="/home/ubuntu/alpinefs/proc" chroot /home/ubuntu/alpinefs /bin/sh

# Change the hostname in the container
hostname container

# Check hostname on host and see that it has not changed as it has in the container
hostname

# Output
rootless-container
```

## Golang Container

- The follow-along video for the code in `main.go` is linked at the bottom
- We will be running shell process in a container using the program as we just have with native Linux commands
- You will need to install [Go](https://go.dev/doc/install) first

```bash
# From the host
go run main.go run /bin/sh

# Output
running [/bin/sh] as user 1000 and in process 10275
running [/bin/sh] as user 0 and in process 1
```

- The program essentially does the same things as above
- It includes several namespace flags that we wish to use for our container in the command built in the `run` function
- The flag to make note of is `syscall.CLONE_NEWUSER`, since we now know that this will not require higher permissions from the host's user
- It also maps the host user and group IDs to the root user and group ID of the container as we did previously
- When the command is executed the first time, it the program within it self by executing `/proc/self/exe` with the command of `child` instead of `run`
- When the program runs that second time, it is doing so within the newly create namespaces, where the root directory is changed to the Alpine filesystem, and `proc` folder in the Alpine filesystem is mounted


```bash
# From the container
ps aux

# Output
PID   USER     TIME  COMMAND
    1 root      0:00 /proc/self/exe child /bin/sh
    6 root      0:00 /bin/sh
    8 root      0:00 ps aux

# From the host
ps -t 1 -f

# Output
UID          PID    PPID  C STIME TTY          TIME CMD
ubuntu      8850    8849  0 13:21 pts/1    00:00:00 bash
ubuntu     10439    8850  0 16:46 pts/1    00:00:00 go run main.go run /bin/sh
ubuntu     10475   10439  0 16:46 pts/1    00:00:00 /tmp/go-build2036034500/b001/exe/
ubuntu     10479   10475  0 16:46 pts/1    00:00:00 /proc/self/exe child /bin/sh
ubuntu     10484   10479  0 16:46 pts/1    00:00:00 /bin/sh
```

- Just as before, the host will see these processes as its own, but the container has the illusion of running as root

## Sources

- [Containers under the Hood](https://linuxera.org/containers-under-the-hood/)
- [Rootless Containers from Scratch](https://www.youtube.com/watch?v=jeTKgAEyhsA)