---
layout: post
title: How to set up SFTP with a jailed user on an AWS EC2 instance
description: A complete guide to setting up SFTP securely on an AWS EC2 instance.
category: tech
excerpt_separator: <!--more-->
comments: true
---

Recently at my work we had a client who wanted to routinely dump data via FTP onto our production server.

We set up a user account for them to use. However, to maximise security, we also wanted to:

* Jail the SFTP user to a single home directory
* Prevent SSH access

What seemed a straightforward process became more complex, mostly due to some UNIX idiosyncrasies with chroot jails and file permissions.
<!--more-->
### Guide

The first step is to create the user that the client will log in as. If you aren't using password authentication, skip the password step.


```
sudo adduser ftp-user
sudo passwd ftp-user
```

The next step is to create the directory the user will be jailed to. This is the tricky bit: in order to chroot a user to a directory, the directory must be owned by root with no write access. The same applies to every directory up the tree.

What's the solution? To create a writable folder owned by the FTP user *inside* the jailed home directory.

Let's create a home directory for our SFTP user and set it to be owned by root:

```
sudo mkdir ftp_home
sudo chown root ftp_home
sudo chmod go-w ftp_home
sudo usermod -d ftp_home ftp-user
```

Now let's create the folder to house the FTP data and give our user read and write access:

```
sudo mkdir ftp_home/data
sudo chown ftp-user ftp_home/data/
sudo chmod ug+rwX ftp_home/data/
```

If you're planning to add the connecting user's public key rather than using a password, set up the FTP user's authorised keys:

```
mkdir ftp_home/.ssh
chmod 700 ftp_home/.ssh
touch ftp_home/.ssh/authorized_keys
chmod 600 ftp_home/.ssh/authorized_keys
```

Now we need to enable SFTP and jail our user. Append the following to your `sshd_config`. For me it was found at `/etc/ssh/sshd_config`. Delete the password authentication line if you are aren't using password auth.

```
Match User ftp-user
    PasswordAuthentication yes
    ChrootDirectory /var/app/shared/ftp_home
    X11Forwarding no
    AllowTcpForwarding no
    ForceCommand internal-sftp
```

Dont forget to restart SSH afterwards:

```
sudo service sshd restart
```

Voilà! We have a jailed SFTP user. The only thing left is to deny SSH access. You can do this by changing their shell to a no login shell. I used:

```
sudo chsh -s /sbin/nologin ftp-user
```

And that's it. Now you can your let someone transfer files to your server with peace of mind.
