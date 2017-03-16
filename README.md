# **Setup Gitlab on Windows using CentOS-7 as Guest VM in VirtualBox**

If you want to run gitlab, you will need a linux/unix machine. Gitlab can't be installed on Windows (as of now). One option is to run Gitlab on a Linux VM running on a Windows. Below are the steps to perform to install gitlab on CentOS-7 using vitualbox guest vm on a windows host.

Useful Links:

- The official installation instructions are [here][1].
- The official documentation is [here][2]

## Pre-installation Steps

### Install VirtualBox
First install virtualbox on your host windows machine.
Reference: <https://wiki.centos.org/HowTos/Virtualization/VirtualBox/CentOSguest><br>
Download CentOS-7 minimal 64 bit iso from [here][1]

- Set virtual optical disk to point to the location of downloaded .iso.
- Set NAT and forward the required ports. In our example, we will forward the following ports from host -> guest: 2122 -> 22, 9443 -> 443, 9180 -> 80. This means that your git clients will use port 2122 for ssh, 9443 for https and 9180 for http. You can also use a bridged network, but you will have to address security hardening, firewall etc.
- Now start virtual box and go thru installation process. Make sure you meet minimum hardware requirements for gitlab.

After the VM starts, network will not work with default settings. Follow these steps for enable access to the internet. [Reference][1]

Edit ifcfg-enp0s3 and change ONBOOT=yes, like so:

```bash
cd /etc/sysconfig/network-scripts/
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
```

Helpful: On your host (in `C:\Windows\System32\drivers\etc\`) Add localhost to file hosts for 127.0.0.1:

```
127.0.0.1 localhost
::1             localhost
```

Optionally use a shell terminal (like git bash) on your windows host, to ssh using the root user and password, for easy copy paste. It will save you from typing all the commands in the virtualbox guest vm.

```bash
ssh -p 2322 root@localhost
```

If you get man-in-the-middle attack warnings when accessing the virtualbox guest vm, old key entries, clear the relevant entry in `~/.ssh/known_hosts`. When you access it for the first time, the shell will ask you to confirm and add the correct entry again.

Optionally you can add the following entry in .ssh/config on local to make it easier to ssh into the vm.

```
Host gitlab-root
   HostName localhost
   port 2322
   User root
```

So now you can use the following command to ssh into the virtualbox guest vm

```bash
ssh gitlab-root
```

### Create Admin User Account:

We will use the account gitadmin with sudo access to install gitlab. [Reference][1]

```bash
adduser gitadmin
passwd gitadmin
usermod -aG wheel gitadmin
```

Now exit out of root, and ssh into the machine using the new account gitadmin

```
ssh -p 2322 gitadmin@localhost
```

## Setup SSH Access Using Keys:

[Reference][1]

1. Generate keys on the host

  ```bash
  cd ~/.ssh
  ssh-keygen -t rsa -C "Gitlab" -b 4096
  # save files with a unique name: ~/.ssh/id_rsa_gitadmin and without entering password
  ```

2. On the virtualbox guest vm, open ssh via terminal and login as gitadmin

  ```bash
  mkdir .ssh
  chmod 700 .ssh
  exit
  ```

3. Copy the newly generated public key from the host to the virtualbox guest vm. So back on the host:

  ```bash
  cat ~/.ssh/gitlab3/id_rsa_git.pub | ssh -p 2322 gitlab3 'cat >> .ssh/authorized_keys'
  ```

4. On the virtualbox guest vm, change permissions:

  ```bash
  chmod 600 .ssh/authorized_keys
  ```

5. Optionally, on the host, edit .ssh/config and add the following entry:

  ```
  Host gitadmin
     HostName localhost
     port 2322
     User git
     RSAAuthentication yes
     PreferredAuthentications publickey
     IdentityFile ~/.ssh/gitlab3/id_rsa_gitadmin
  ```

6. Let's test ssh using keys (without password)

  ```bash
  ssh gitadmin
  ```

### Hardening CentOS-7

Optionally you may want to consider hardening your virtualbox guest vm. This is beyond the scope of this article. But you can follow this [reference][1]

- Disable root login
- Limit user logins
- Disable protocol 1
- Use non-standard ssh port
- Use public & private keys

## Install Gitlab Omnibus

Our virtualbox guest vm is now ready to install the required dependencies for gitlab omnibus:

```bash
sudo yum check-update
sudo yum update
sudo yum install curl policycoreutils openssh-server openssh-clients
sudo yum install libsemanage-static libsemanage-devel
sudo systemctl enable sshd
sudo systemctl start sshd
sudo systemctl enable postfix
sudo systemctl start postfix
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld

# Optional
sudo yum install deltarpm
```

Now, we are ready to install gitlab omnibus.

```bash
# You can download the script and install it if you need to.
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo yum install gitlab-ce

# Let's keep all defaults and configure gitlab
sudo gitlab-ctl reconfigure
# You should see something like so:
    run: logrotate: (pid 637) 87s; run: log: (pid 636) 87s
    run: nginx: (pid 642) 86s; run: log: (pid 641) 86s
    run: postgresql: (pid 658) 86s; run: log: (pid 657) 86s
    run: redis: (pid 644) 86s; run: log: (pid 643) 86s
    run: sidekiq: (pid 640) 86s; run: log: (pid 639) 86s
    run: unicorn: (pid 656) 86s; run: log: (pid 655) 86s
```

Let's test the default installation. Assuming that in virtualbox you forwarded an unused port (8180) to 80 in the virtualbox guest VM. Open the following URL in a browser:<br>
<http://localhost:8180/users/sign_in><br>
It should ask you to set up admin access to the gitlab application (the user is root, default password is password)

### Setup File Sharing in VirtualBox
See [here](file-sharing.md)

[1]: https://wiki.centos.org/HowTos/Network/SecuringSSH
[2]: https://docs.gitlab.com/omnibus/README.html

## Configure Gitlab

Reference documentation is here: <https://docs.gitlab.com/omnibus/settings/configuration.html>

Let us configure gitlab to allow ldap access, enable https only and disable signup. Note that all gitlab config for the omnibus installation is handled via this file here: `/etc/gitlab/gitlab.rb`<br>
Reference: <https://gitlab.com/gitlab-org/omnibus-gitlab/blob/7-8-stable/README.md>

We will create a local copy of gitlab.rb on the host and scp it onto the virtualbox guest vm. But before we do this, let us create self signed certs on the virtualbox guest vm.

### NGINX Config
See [nginx config](nginx.md)

### Enable LDAP Authentication Config
See [ldap config](ldap.md)

### SMTP - Email Config
Reference: https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/smtp.md

### Backups Config
See [backup config](backup.md)

### Logs Config
Reference: https://docs.gitlab.com/omnibus/settings/logs.html

We will use runit logs. use these settings in gitlab.rb:
```
# runit logs
logging['svlogd_size'] = 100 * 1024 * 1024 # rotate after 200 MB of log data
logging['svlogd_num'] = 30 # keep 30 rotated log files
logging['svlogd_timeout'] = 24 * 60 * 60 # rotate after 24 hours
logging['svlogd_filter'] = "gzip" # compress logs with gzip
logging['svlogd_udp'] = nil # transmit log messages via UDP
logging['svlogd_prefix'] = nil # custom prefix for log messages
```

You can check nginx logs here:
```bash
sudo gitlab-ctl tail nginx
```
### Maintenance Information
See [maintenance information](maintenance.md)

### Customize Homepage

You can change appearence of the home screen in admin > appearence section (root login). Below is an example md:

```
Welcome to Gitlab for the Department of Revenue, KY. Gitlab is open source software to collaborate on code. Manage Git repositories with fine-grained access controls that keep your code secure. Perform code reviews and enhance collaboration with merge requests. Each project can also have an issue tracker and a wiki.  
Use the 'AD Account' tab to sign in using your revenue account. Do **NOT** prefix FIN to your User ID.
```

If you need further customizations: <https://kovah.me/en/customize-gitlab-installation/>
