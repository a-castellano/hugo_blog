---
title: "Deploying VM's using cloud-init"
date: 2020-05-31T23:09:07+02:00
draft: false
---

# The beginning - VM templates

Instead of having a server with multiple services installed, almost three years ago I started to use larger servers with KVM in order to manage multiple VM for those services. I had three "template" machines, for *Debian 9*, *Ubuntu 16.04* and *Ubuntu 18.04*. Each time I needed a new machine I followed the following steps:
* Clone the "template" VM.
* Power on the new VM.
* Reconfigure network config and DNS's in order to change VM network interface.
* Reboot the machine.
* Hope that I did not fail in any step.
* Start working with the new VM.

After 5-10 minutes my new VM was ready, great....

Really? Well, let's face it, after two years doing this I've got tired of deploying VM in that way. 

Also, in order to save time for next VM deploys I often update those templates (basic tools and kernel). At the end I had another three VM's to manage.

# Proxmox Clusters

Since last year my server has been running out of free space disk. I decided to rent a new server and start using [Proxmox](https://www.proxmox.com/en/) instead of managing raw KVM servers.

Cluster is working and it works great. Now let's create new VM's templates.... not again. I will end cloning VM's again. Let's try another way.

# Cloud-init, (maybe) this is the way

What Cloud-init is? According with [official documentation](https://cloudinit.readthedocs.io/en/latest/) Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialization. It allows us to ensure certain configurations are present on boot time on our VM's such as some interesting things I'm interested about:
* Network configuration
* User management
* Repository management

Those three items would allow me getting rid of maintaining templates. So let's begin.

## Cloud images

We need to stop using current Ubuntu server images and use (Ubuntu Cloud Images)[https://cloud-images.ubuntu.com/] which include Cloud-init configs. We will make changes in order to set Network configuration, users and repos.

Download latest Ubuntu Bionic Cloud image
```bash
wget -O /var/lib/vz/bionic.img https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img
```

In order to edit Cloud-init images we need to install libguestfs-tools
```bash
apt-get install libguestfs-tools
```

Now is time to edit Cloud-init config:
```bash
virt-edit -a /var/lib/vz/bionic.img /etc/cloud/cloud.cfg
```


Let's review how I have changed the original file:

## Cloud-init levels

The following configuration will be the default one but it can be updated without changing this default configuration. We can add a Cloud-init device that can override this configuration. For example, we can change hostname or network config when creating new machine form our template.
## Cloud-init modules

Here are the modules that Cloud-init will load in order to make changes, here are only listed few of them, the ones I'm using or u pretend to use in the near future.

```yml
cloud_init_modules:
 - write-files
 - growpart
 - resizefs
 - disk_setup
 - mounts
 - set_hostname
 - update_hostname
 - update_etc_hosts
 - ca-certs
 - users-groups
 - ssh

cloud_config_modules:
 - emit_upstart
 - locale
 - set-passwords
 - grub-dpkg
 - apt-pipelining
 - apt-configure
 - timezone
 - runcmd

cloud_final_modules:
 - package-update-upgrade-install
 - salt-minion
 - scripts-vendor
 - ssh-authkey-fingerprints
 - keys-to-console
 - final-message
```

### Disable root and preserve hostname

Root cannot log in directly in our server, also hostname can be altered by Cloud-init config.

```yml
disable_root: true

preserve_hostname: false
```

### SSH public keys

All created user will have the following ssh public keys authorized:
```yml
ssh_authorized_keys:
  - ssh-rsa MyPublicSSH Key
```

### User config

```yml

Our user **youruser** will be able to become root accrding with *sudo* given option:

system_info:
   # This will affect which distro class gets used
   distro: ubuntu
   # Default user name + that default users groups (if added/used)
   default_user:
     name: youruser
     lock_passwd: True
     gecos: Youruser
     groups: [adm, audio, cdrom, dialout, dip, floppy, lxd, netdev, plugdev, sudo, video]
     sudo: ["ALL=(ALL) NOPASSWD:ALL"]
     shell: /bin/bash
   # Automatically discover the best ntp_client
   ntp_client: auto
   # Other config here will be given to the distro class and/or path classes
   paths:
      cloud_dir: /var/lib/cloud/
      templates_dir: /etc/cloud/templates/
      upstart_dir: /etc/init/
   ssh_svcname: ssh

password: defauluerpassword
```

Wait, are you posting a clear text password in your Cloud-init confg? Why? Is it necessary?

Yes, password has to be in clear text. No, it isn't necessary. Why am I setting a password? Because sometimes accidents happen an machines lost internet connection, I would like to enter into my VM using hypervisor screen. Of course we will set a sshd config that disallows password authentication.ººº:w


### Timezone and locale config

```yml
timezone: "Europe/Madrid"
locale: en_US
```

### SSH config

Cloud-init allows us to place files in our system, let's change default sshd config. Only logging using ssh keys is allowed there is only one user allowed to connectCloud-init allows us to place files in our system, let's change default sshd config. Only logging using ssh keys is allowed there is only one user allowed to connect.

```yml
write_files:
- content: |

    Port 22
    Protocol 2
    # HostKeys for protocol version 2
    HostKey /etc/ssh/ssh_host_rsa_key
    HostKey /etc/ssh/ssh_host_dsa_key
    HostKey /etc/ssh/ssh_host_ecdsa_key
    HostKey /etc/ssh/ssh_host_ed25519_key
    #Privilege Separation is turned on for security
    UsePrivilegeSeparation yes

    # Lifetime and size of ephemeral version 1 server key
    KeyRegenerationInterval 3600
    ServerKeyBits 1024

    # Logging
    SyslogFacility AUTH
    LogLevel INFO

    # Authentication:
    LoginGraceTime 60
    PermitRootLogin prohibit-password
    StrictModes yes
    PasswordAuthentication no
    PubkeyAuthentication yes
    RSAAuthentication yes
    PubkeyAuthentication yes

    AllowUsers youruser

    IgnoreRhosts yes
    RhostsRSAAuthentication no
    HostbasedAuthentication no

    PermitEmptyPasswords no

    ChallengeResponseAuthentication no

    X11DisplayOffset 10
    PrintMotd no
    PrintLastLog yes
    TCPKeepAlive yes

    AcceptEnv LANG LC_*

    Subsystem sftp /usr/lib/openssh/sftp-server

    UsePAM yes

  path: /etc/ssh/sshd_config
  owner: root:root
  permissions: '0644'
```

### Apt config

My VM's do not use official Ubuntu repository lists, they use two repos:
* **repo-bionic.windmaker.net** which stores a replica of official Ubuntu packages.
* **repo.daedalus-project.io** which stores Daedalus Project packages and other packages like Percona releases.

First, we need to disable all Ubuntu repos first using *disable_suites* directive. After that we set our repo URL's and their GPG keys.

```yml
apt:
  disable_suites: [updates, backports, security, proposed, release]

  sources:
    repo-bionic.list:
      source: "deb http://repo-bionic.windmaker.net/ bionic main"
      key: |
        -----BEGIN PGP PUBLIC KEY BLOCK-----

        mQGNBFsf464BDACpTuXB5TO1Xaca2UfI9JeO2PmYOSKtnM6Fm+xkhP/XwM1+NIOG
        guvw/+g3JVmUN6cMklVDhUgE46iVn7OccXIKYoKFRWLFRkPx7KGFdxA2bxQ/N6zt
        TyfxUyskRQPvDbbBX/ExGsVl/Y+1hPullJlvuXja6pUXIa/q0I+/WYu6rlnDlp6Y
        SuTVlGlQY2TJ7ActOsOAO+TxkLonn/NJRPmJUmZW8foWgPMDgIIj/f800os9iFjx
        89cykmHgDc3Lt/9YE/OTfvrirmqKdlatcb+S9zAJ00F9/w+30qh+Y9p/IzgbxF6j
        ebuW8Ag6zAlvNgQZmYigbeZJoUZtBd9cM5Tm5gQZRoyCy+Vf00dl2Acl2OZ84OGA
        L3zrkeb9Mz40/FazEZsMpr2TtVQmYKRY8NRGX8RULxaePtagDos2nb7Em/MAVyI5
        ar2/Zh0wYxbRhjvBfP0gn97lprov72Ce8F/GoP8K798t5b0RoHtXQdarBO+Z7KuC
        CbsIAXNA5Zl25gMAEQEAAbQ6w4FsdmFybyBDYXN0ZWxsYW5vIFZlbGEgPGFsdmFy
        by5jYXN0ZWxsYW5vLnZlbGFAZ21haWwuY29tPokB1AQTAQoAPhYhBH0JhQsOihgj
        KUwRwfiEJpJagjkJBQJbH+OuAhsDBQkDwmcABQsJCAcCBhUKCQgLAgQWAgMBAh4B
        AheAAAoJEPiEJpJagjkJzEUL/1IchCrksbt95ngsWpfBU9pP69xSlGoJwQKWBycy
        uShkHCtH5F6DNRYVfFNbvgPaH2vvZrtKNzZqGfhJbWRvR1owdF4XOz0tPHTcU3NY
        MJT0KfRh8rms8Ai/jDdb9OY2kReqddUAyLLjdfd0NC+g5DYF5zagliEpkxWYAi/0
        H6tpB0Rg25L0Iu/M/toIu6VSCZqVRTfggrPKWddaA/kGGmbHKyAsfLEXkwXIT2IK
        N7nXNJCGCWrgmWtCs/8VVofgqspOoyU8ZxfsKrvlA+yEi53iOuDcLOycw6uRpjX9
        F93T2Vvt0bjPq85cViABZHkwXXUbRwxD9GOLVmGD+e7S6Vbw04mJff3phZ6x/AIP
        iT8PZOUSbngvS1SnKOfK1dSvwkoeMhV1xn2NH7lJPQmMwh3o3AVLGidIRfkegduK
        iG49Byn7qVhQu/6jKEyWHS2lMw0uKa34MRRsYtqqreUCp2F/51vSrJxxIgLF2pM6
        df/QgV3fNq9tEuP0TE52OSPx6rkBjQRbH+OuAQwA4bcfEDVDLyywSREfFdLuK2+E
        7oPLTGVw6T4fKhsR47hSydOMXC7RhLXKIUjZX2IL1u0gNzVJQuCkbHkSxdr6rwsc
        IjIuWgkzLfSw5z1BzHjJbfxS32XhQdM4noFx7KvFjh2Umwz1SPWvcj4Az1uX0kbL
        gsK5eJQ1YBz2lMdhZKMLNMZSsNnHuxXg6EfpfUHDusiTBBdttO2kQjl3eC4wZaOc
        VPMYO2HjeWRaK9Vhvt8aF/PU0bijBZZ69ZSv/E9wjgFlgvPsvMA92WNCawnC0JwW
        kosL+qjbn43EldTZe+RWTbgzW/e9bvCOUpeWlHKl1pncccM0LPWAGZi/HKQUFIds
        AyfJLK4Vr0p1s9p6TghiF0RVlvwWdRHhfRTLjASrHuVbh+eth7LC6/ZHhRrVdZTa
        qfSYNfvW0nUH/uF9XBr21XWTn67i8KiVDuveAPebq+R+SwVidWpin3t9U6ykODxc
        2gjIL25nRvZuptb0hHIm5OCnW8yi5VO3TLKZQBWlABEBAAGJAbwEGAEKACYWIQR9
        CYULDooYIylMEcH4hCaSWoI5CQUCWx/jrgIbDAUJA8JnAAAKCRD4hCaSWoI5CbqJ
        C/oDT5G9HGBDkfxv35ohU8p/J2tnOBmhKGvCLfIsK068bW7F/zX+P72yEY6QwT+Y
        IPC2cnfwhJ31Pd8muYhn8b7MJwSNdR0nxA3ORmhjMYGlbq4jH32RLvU9Qk6JA+Oh
        Xyw9uglilp454BLqe3ew+PxerN8nZKvVTwlmfZmI15N7PG3DP18/P8r73EUR6+ud
        F27vELLmgoqShOYRvh+yLsOcxPQlV5eCY5rcu0avATtuFrYpy02jTk5/ABMaMr5V
        /RooRtH3L5iueM2fdg5JIgyQ1cILPbsW+v53OIthjHNfgdIXGQO3svzA8U1/5qJk
        oghFyg6SiG/2+bdo2ociPGMu9VH/0o3Z9+YRrKeNJIwe1rYeWH/L8qJQ3kjWFcpB
        OfY50O2FiL1O24M8nOH5Ub3bdCH8+kW4SRQsUCsbnSIQzPaDJyO6vJ5myHh8B6IY
        ffYSW0AkNsBCjlAysc0w3X/CeuW0yVGg4PQUWehVoBvob1/08DvGHiq285hBftoB
        Gb4=
        =Uzuz
        -----END PGP PUBLIC KEY BLOCK-----

    repo-daedalus-project.list:
      source: "deb [arch=amd64] http://repo.daedalus-project.io/ bionic main"
      key: |
        -----BEGIN PGP PUBLIC KEY BLOCK-----

        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
        eZiQKhRY1gf+exPCSjnji7snbdsSqX0=
        =FDFJ
        -----END PGP PUBLIC KEY BLOCK-----
```

Entire Cloud-init file can be found [here](https://gist.github.com/a-castellano/49cee1dbf1f1e29c81d44536d98189a2).

## Creating a template VM

After editing our Cloud-init file, let's create a VM template using Proxmox.

The new template will be named **bionicTmeplate**. We will set a net card attached to our VM's network, in my case vmbr1. RAM is set to 2048Mb
```bash
qm create 100 --memory 2048 --net0 virtio,bridge=vmbr1 --name=bionicTmeplate
```
Import Ubuntu Bionic Cloud image:
```bash
qm importdisk 100 /var/lib/vz/bionic.img local
```

Attach the new disk to our VM:
```bash
qm set 100 --scsihw virtio-scsi-pci --virtio0 local:100/vm-100-disk-0.raw
```
We will need to add a cloudinit disk too:
```bash
qm set 100 --ide2 local:cloudinit
```
Set first disk as bootable one:
```bash
qm set 100 --boot c --bootdisk virtio0
```
Set the screen using a serial socket, this config will allow us to copy text from hipervisor screen:
```bash
qm set 100 --serial0 socket --vga serial0
```
Finally, we create a template from this VM.
```bash
qm template 100
```

## Creating a new VM

Time to create a new VM using this new template:
```bash
qm clone 100 123 --name Test
```

Let's resize disk before start the VM:
```bash
qm resize 123 virtio0 +30G
```
Each VM must have its own IP address, we config it now:
```bash
qm set 123 --ipconfig0 ip=10.XXX.YYY.10/16,gw=10.XXX.1.11
qm set 123 --nameserver 10.XXX.ZZZ.10
```

Time to start the VM, on startup we will see how our config is applied:

Our network config is applied:
```
         Starting Initial cloud-init job (pre-networking)...
[   21.020495] cloud-init[593]: Cloud-init v. 19.4-33-gbb4131a2-0ubuntu1~18.04.1 running 'init-local' at Sun, 31 May 2020 17:49:26 +0000. Up 20.11 seconds.
[  OK  ] Started Initial cloud-init job (pre-networking).
[  OK  ] Reached target Network (Pre).
         Starting Network Service...
[  OK  ] Started Network Service.
         Starting Network Name Resolution...
         Starting Wait for Network to be Configured...
[  OK  ] Started Network Name Resolution.
[  OK  ] Reached target Host and Network Name Lookups.
[  OK  ] Reached target Network.
[  OK  ] Started Wait for Network to be Configured.
         Starting Initial cloud-init job (metadata service crawler)...
[   24.148124] cloud-init[701]: Cloud-init v. 19.4-33-gbb4131a2-0ubuntu1~18.04.1 running 'init' at Sun, 31 May 2020 17:49:30 +0000. Up 23.82 seconds.
[   24.158206] cloud-init[701]: ci-info: ++++++++++++++++++++++++++++++++++++++Net device info++++++++++++++++++++++++++++++++++++++
[   24.162608] cloud-init[701]: ci-info: +--------+------+------------------------------+-------------+--------+-------------------+
[   24.167279] cloud-init[701]: ci-info: | Device |  Up  |           Address            |     Mask    | Scope  |     Hw-Address    |
[   24.171305] cloud-init[701]: ci-info: +--------+------+------------------------------+-------------+--------+-------------------+
[   24.174399] cloud-init[701]: ci-info: | ens18  | True |        10.XXX.YYY.10         | 255.255.0.0 | global | ce:47:27:a4:12:cb |
[   24.177641] cloud-init[701]: ci-info: | ens18  | True | fe80::cc47:27ff:fea4:12cb/64 |      .      |  link  | ce:47:27:a4:12:cb |
[   24.180768] cloud-init[701]: ci-info: |   lo   | True |          127.0.0.1           |  255.0.0.0  |  host  |         .         |
[   24.184262] cloud-init[701]: ci-info: |   lo   | True |           ::1/128            |      .      |  host  |         .         |
[   24.189674] cloud-init[701]: ci-info: +--------+------+------------------------------+-------------+--------+-------------------+
[   24.194320] cloud-init[701]: ci-info: ++++++++++++++++++++++++++++Route IPv4 info++++++++++++++++++++++++++++
[   24.198817] cloud-init[701]: ci-info: +-------+-------------+-------------+-------------+-----------+-------+
[   24.202828] cloud-init[701]: ci-info: | Route | Destination |   Gateway   |   Genmask   | Interface | Flags |
[   24.206859] cloud-init[701]: ci-info: +-------+-------------+-------------+-------------+-----------+-------+
[   24.209845] cloud-init[701]: ci-info: |   0   |   0.0.0.0   | 10.XXX.1.11 |   0.0.0.0   |   ens18   |   UG  |
[   24.212480] cloud-init[701]: ci-info: |   1   |  10.XXX.0.0 |   0.0.0.0   | 255.255.0.0 |   ens18   |   U   |
[   24.215526] cloud-init[701]: ci-info: +-------+-------------+-------------+-------------+-----------+-------+
[   24.219341] cloud-init[701]: ci-info: +++++++++++++++++++Route IPv6 info+++++++++++++++++++
[   24.221671] cloud-init[701]: ci-info: +-------+-------------+---------+-----------+-------+
[   24.223971] cloud-init[701]: ci-info: | Route | Destination | Gateway | Interface | Flags |
[   24.227114] cloud-init[701]: ci-info: +-------+-------------+---------+-----------+-------+
[   24.230247] cloud-init[701]: ci-info: |   1   |  fe80::/64  |    ::   |   ens18   |   U   |
[   24.233530] cloud-init[701]: ci-info: |   3   |    local    |    ::   |   ens18   |   U   |
[   24.239004] cloud-init[701]: ci-info: |   4   |   ff00::/8  |    ::   |   ens18   |   U   |
[   24.242550] cloud-init[701]: ci-info: +-------+-------------+---------+-----------+-------+
```

Custom repos are added to VM's repo list:
```
Test login: [   35.727835] cloud-init[1090]: Generating locales (this might take a while)...
[   36.341105] cloud-init[1090]:   en_US.ISO-8859-1... done
[   36.342903] cloud-init[1090]: Generation complete.
[   39.693040] cloud-init[1090]: Get:1 http://repo.daedalus-project.io bionic InRelease [5152 B]
[   39.762068] cloud-init[1090]: Get:2 http://repo-bionic.windmaker.net bionic InRelease [3000 B]
[   39.872322] cloud-init[1090]: Get:3 http://repo.daedalus-project.io bionic/main amd64 Packages [70.4 kB]
[   40.040415] cloud-init[1090]: Get:4 http://repo-bionic.windmaker.net bionic/main amd64 Packages [15.2 MB]
[   47.544781] cloud-init[1090]: Fetched 15.3 MB in 5s (2863 kB/s)
[   48.836633] cloud-init[1090]: Reading package lists...
[   48.938858] cloud-init[1090]: Cloud-init v. 19.4-33-gbb4131a2-0ubuntu1~18.04.1 running 'modules:config' at Sun, 31 May 2020 17:49:42 +0000. Up 35.45 seconds.
[   49.690566] cloud-init[1735]: Reading package lists...
[   49.854240] cloud-init[1735]: Building dependency tree...
[   49.857138] cloud-init[1735]: Reading state information...
[   49.951017] cloud-init[1735]: Calculating upgrade...
[   50.062925] cloud-init[1735]: The following package was automatically installed and is no longer required:
[   50.067178] cloud-init[1735]:   grub-pc-bin
[   50.069484] cloud-init[1735]: Use 'apt autoremove' to remove it.
[   50.116744] cloud-init[1735]: The following packages will be upgraded:
[   50.121606] cloud-init[1735]:   init init-system-helpers iproute2
[   50.158913] cloud-init[1735]: 3 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
[   50.161701] cloud-init[1735]: Need to get 812 kB of archives.
[   50.165186] cloud-init[1735]: After this operation, 285 kB of additional disk space will be used.
[   50.168542] cloud-init[1735]: Get:1 http://repo-bionic.windmaker.net bionic/main amd64 init-system-helpers all 1.56+nmu1~ubuntu18.04.1 [38.2 kB]
[   50.193411] cloud-init[1735]: Get:2 http://repo-bionic.windmaker.net bionic/main amd64 init amd64 1.56+nmu1~ubuntu18.04.1 [6040 B]
[   50.212301] cloud-init[1735]: Get:3 http://repo-bionic.windmaker.net bionic/main amd64 iproute2 amd64 4.18.0-1ubuntu2~ubuntu18.04.1 [768 kB]
```

Finally our public key is added to **youruser** authorized keys:
```
[   54.253006] cloud-init[1735]: Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
ci-info: +++++++++++++++++Authorized keys from /home/ventus/.ssh/authorized_keys for user ventus+++++++++++++++++
ci-info: +---------+-------------------------------------------------+---------+--------------------------+
ci-info: | Keytype |                Fingerprint (md5)                | Options |         Comment          |
ci-info: +---------+-------------------------------------------------+---------+--------------------------+
ci-info: | ssh-rsa | 50:12:39:be:84:c1:f1:b6:89:6c:43:85:40:36:b1:f5 |    -    | yoursshid@yourhost.local |
ci-info: +---------+-------------------------------------------------+---------+--------------------------+
```

## Testing VM access

Let's login into our VM and check all our configs are working:
```
$ ssh ventus@10.XXX.YYY.10
Warning: Permanently added '10.XXX.YYY.10' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun May 31 20:24:51 CEST 2020

  System load:  0.81              Processes:            84
  Usage of /:   3.4% of 31.04GB   Users logged in:      0
  Memory usage: 6%                IP address for ens18: 10.XXX.YYY.10
  Swap usage:   0%

0 packages can be updated.
0 updates are security updates.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ventus@Test:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            985M     0  985M   0% /dev
tmpfs           200M  616K  199M   1% /run
/dev/vda1        32G  1.1G   30G   4% /
tmpfs           997M     0  997M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           997M     0  997M   0% /sys/fs/cgroup
/dev/vda15      105M  3.6M  101M   4% /boot/efi
tmpfs           200M     0  200M   0% /run/user/1000
ventus@Test:~$ sudo su
root@Test:/home/ventus# ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 6e:38:1b:41:f6:7a brd ff:ff:ff:ff:ff:ff
    inet 10.XXX.YYY.10/16 brd 10.XXX.255.255 scope global ens18
       valid_lft forever preferred_lft forever
    inet6 fe80::6c38:1bff:fe41:f67a/64 scope link
       valid_lft forever preferred_lft forever
root@Test:/home/ventus# cat /etc/apt/sources.list.d/repo-bionic.list
deb http://repo-bionic.windmaker.net/ bionic main
root@Test:/home/ventus# cat /etc/apt/sources.list.d/repo-daedalus-project.list
deb [arch=amd64] http://repo.daedalus-project.io/ bionic main
```

## Final steps

After setting up the VM we should remove Cloud-Init device:
```bash
qm set 123 --ide2 none
```
