---
title: "Docker_privesc"
date: 2021-05-05T10:06:52+02:00
---

Yesterday I was playing with [linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) and found a privilege escalation path on my system. 

My user was automatically added to the `docker` group during the installation of docker. That means that my user has write access to `docker.sock` and can mount the local file system on a container to gain root access.

Running this command is all it takes to become root: `docker -H unix:///var/run/docker.sock run -v /:/host -it image chroot /host /bin/bash`

A possible remediation could be to create a dedicated user to use docker.

More here: https://book.hacktricks.xyz/linux-unix/privilege-escalation#writable-docker-socket