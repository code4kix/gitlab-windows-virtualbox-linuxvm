# Setup File Sharing in VirtualBox

If you are running gitlab on a virtualbox guest vm, you may want to set up file sharing between your host and virtualbox guest vm for backing up your gitlab data.

**Install guest additions on linux**  
Reference: <https://wiki.centos.org/HowTos/Virtualization/VirtualBox/CentOSguest>

To retain guest additions after kenel update you need dkms. But it is available in a 3rd party repo. So let's first install the repo. Reference: <http://lifeofageekadmin.com/how-to-install-virtualbox-5-additions-on-centos-7/>

```bash
sudo yum install epel-release
sudo yum install dkms
```

TODO Install guest additions on CentOS-7 in command mode
Reference: <http://unix.stackexchange.com/a/18926/180667>

```bash
# sudo yum update
# Error prestodelta not available - TODO fix later

# Note that we don't need to install all the dev tools. Install only the following
sudo yum install gcc make kernel-devel bzip2

# Before the next step, you have to insert guest additions disk in vitualbox on the host using virtualbox gui
# click on devices > insert guest additions
# Now you can mount it in your guest machine:
sudo mkdir -p /media/cdrom
# CentOS-7 changes /dev/scd0 to sr0
sudo mount /dev/sr0 /media/cdrom
# sh /media/cdrom/VBoxLinuxAdditions.run
```

On host, create a dir and share (using shell or vbox gui) Reference: <https://nakkaya.com/2012/08/30/create-manage-virtualBox-vms-from-the-command-line/>

On the host, open bash terminal:

```bash
cd 'C:\gitlab\'
# Example share "C:\gitlab\backups"
mkdir backups
# On windows, VBoxManage is here: "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
# VBoxManage sharedfolder add <vmname> --name <some_share_name> --hostpath <hostpath>
cd "C:\Program Files\Oracle\VirtualBox"
# for shell use:
./VBoxManage sharedfolder add CentOS-7-Gitlab --name backups --hostpath "C:\gitlab\backups" --automount
# for command use:
# VBoxManage sharedfolder add CentOS-7-Gitlab --name backups --hostpath "C:\gitlab\backups" --automount
```
Now, ssh into the virtualbox guest and run the following:
```bash
# now you are ready to mount your share - note backups should match above step
mkdir -p /home/gitadmin/gitlab/backups
sudo mount -t vboxsf backups /home/gitadmin/gitlab/backups
```

**If you have to umount the shares:**  

Remove it in the host:
```bash
# on the host, run the following:
./vboxmanage sharedfolder remove "CentOS-7-Gitlab" --name backups
```
Optionally, clean-up on the virtualbox guest:
```bash
# ssh into the guest and run the following to remove the share:
rmdir /home/gitadmin/gitlab/backups
```
