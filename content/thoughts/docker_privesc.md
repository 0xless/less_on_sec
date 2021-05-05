---
title: "Docker_privesc"
date: 2021-05-05T10:06:52+02:00
---

Yesterday I was playing with [linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) and found a privilege escalation path on my system.  
It was on `docker.sock`,  a socket that basically allows a user to run docker. While it is fundamental that it's there, it should be secured with the right permissions! 

My user was automatically added to the `docker` group during the installation of docker and that meant that I could mount my local file system on a container with root privileges to gain root access.

Running this command is all it takes to get root access: `docker -H unix:///var/run/docker.sock run -v /:/host -it image chroot /host /bin/bash`

More here: https://book.hacktricks.xyz/linux-unix/privilege-escalation#writable-docker-socket

