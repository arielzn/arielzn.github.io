---
layout: post
title:  "Remote shares"
date:   2016-08-03 02:06:07 +0200
categories: ssh rsync
---


### rsync through SSH

Ideally, ssh access to the server should be defined in the `~/.ssh/config` file, 
specially if you have custom ports and/or keypairs, this will avoid cluttering the command.
Having that defined, the rsync with a folder on a remote share is done with:

```bash
rsync  -avz --progress -u /origin/path  user@server:/dest/path  
```

### Mount remote share with SSH access

The same is true here for setting server access on `~/.ssh/config`.
The most practical is to set this on the local-machine's fstab:

```bash
remoteuser@server:  /local/mountpoint fuse.sshfs noauto,_netdev,user,idmap=user,transform_symlinks,allow_other,default_permissions,uid=local_userid,gid=local_groupid 0 0
```

This will take care of translating remote user and group's ID to the local ones, to provide a smooth mount for local edition of files/folders.
The local mount point `/local/mountpoint` must be owned by the local user.
It won't mount at boot (due to `noauto` option) but only on-demand by the user doing: 

```bash
mount /local/mountpoint
```

You must add the `user_allow_other` option to `/etc/fuse.conf` to be able to mount and unmount without root.
