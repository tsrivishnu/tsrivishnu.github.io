---
layout: default
comments: true
tags: [Ruby on rails, Docker for mac]
title: Running Ruby on rails app in docker on MacOS
---

For a long time, I have avoided developing our dockerised Ruby on rails app
on MacOS because of the bad performance with volume mounts.
However, I want MacOS for everything else I do so, I decided to use a Mac as
my primary work machine and run an Ubuntu virtual machine managed with Vagrant.

This setup works well for me since I'm confortable with working entirely in
terminal and VIM via SSH.
I can't expect the same from everyone in the team.
Everyone should have the freedom to chose the operating system and their tools.

So, I decided to figure out the setup for developing our Rails on rails app in
Docker for mac.
There are couple of things I had to change to make this work well and I
summarised them in this article.

## 1. Improving file sync performance with native NFS

Docker for Mac has been well known for its issues with file syncing.
Its very slow.
There are different ways to improve this performance.
The popular and most commonly known are:
  1. Using [`osxfs-caching`](https://docs.docker.com/docker-for-mac/osxfs-caching/#tuning-with-consistent-cached-and-delegated-configurations) with the consistency options like `cached`, `delegated` etc.
  2. Separate tools like [`docker-sync`](http://docker-sync.io)
  and [`docker-bg-sync`](https://github.com/cweagans/docker-bg-sync)

I tried the option 1 with consistency options, its better but not good enough.
Before going with option 2, I wanted to research options that don't require
additional tools.
Luckily, I found [this article](https://www.jeffgeerling.com/blog/2020/revisiting-docker-macs-performance-nfs-volumes)
that showed performance benchmarks for tools like `docker-sync` compared with a
natively available alternative: NFS.

However, setting it up needs a little effort initially.
Here are the steps that are involved
1. Allow MacOS to share any directory in the home directory via NFS.
To do so, you need to create or edit the `/etc/exports` file
  ```
  sudo vi /etc/exports
  ```
  and add the following line:
  ```
  /System/Volumes/Data -alldirs -mapall=501:20 localhost
  ```
    * `501` corresponds to your user id. Double check it with `id -u` in your console.
    * `20` corresponds to the user's group id. Double check it with `id -g`.
2. Tell NFS daemon to allow connections from any post.
I guess, we don't really know which post docker will use and so this is
necessary to allow Docker to connect via NFS.

    Edit the following file

    ```
    $ sudo vi /etc/nfs.conf
    ```
    and add the following line.
    ```
    nfs.server.mount.require_resv_port = 0
    ```
3. Restart `nfsd`
```
$ sudo nfsd restart
```

2. Getting SSH agent to work for Capistrano deployments
