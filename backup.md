# **Gitlab Omnibus - Backup Documentation**

Reference: https://docs.gitlab.com/omnibus/settings/backups.html#backup-and-restore-omnibus-gitlab-configuration

Assuming that you set up a file share between the host and the virtualbox guest vm (Refer to #File-Share.md to see how), read on.

**Note:**
* Separate config, security and application data backups, don't keep them all together. If you separate your configuration backup from your application data backup, you reduce the chance that your encrypted application data will be lost/leaked/stolen together with the keys needed to decrypt it.
* When you restore, gitlab versions must match.

To schedule automatic backup jobs using cron, install cronie on the virtualbox guest vm:
```bash
# install cron
sudo yum install cronie
```

There are three separate sets of files to backup: configuration data, security data and application data.

After you have already setup a backup share in virtualbox and mounted it correctly, do the following steps:

**1. Config Data Backup**  

For omnibus installations, configuration data is here: `/etc/gitlab`.  
To create a daily config data backup, edit the cron table for user root:
```bash
# create config directory in the file-share for storing config data backup
mkdir ~/gitlab/backups/config
# open cron table
sudo crontab -e -u root
```
The cron table will appear in an editor.

Enter the command to create a compressed tar file containing the contents of /etc/gitlab/. For example, schedule the backup to run every morning after a weekday, Tuesday (day 2) through Saturday (day 6):
```
15 04 * * 2-6  umask 0077; tar cfz /secret/gitlab/backups/config/$(date "+etc-gitlab-\%s.tgz") -C / etc/gitlab
```
Cron is rather particular Note the following:
1. The empty line after the command
2. The escaped percent character: \%

**To manually perform a backup:**
```bash
# cd into the directory that you wish you save the backup file to, for example:
cd ~/gitlab/backups/config
# Example backup command for /etc/gitlab:
# Create a time-stamped .tar file in the current directory.
# The .tar file will be readable only to root.
sudo sh -c 'umask 0077; tar -cf $(date "+etc-gitlab-%s.tar") -C / etc/gitlab'
```

**To Restore:**  
You can extract the .tar file as follows.

```bash
# Rename the existing /etc/gitlab, if any
sudo mv /etc/gitlab /etc/gitlab.$(date +%s)
# Change the example timestamp below for your configuration backup
sudo tar -xf etc-gitlab-1399948539.tar -C /
```

Remember to run sudo gitlab-ctl reconfigure after restoring a configuration
backup.

**2. Security Keys Backup**

Your machines SSH host keys are stored in a separate location at `/etc/ssh/`. Let's backup and restore those keys to avoid man-in-the-middle attack warnings if you have to perform a full machine restore.

Note that this is a one-time back up. There is no need to schedule this using cron.

```bash
# cd into the directory that you wish you save the backup file to, for example:
mkdir -p ~/gitlab/backups/security
cd ~/gitlab/backups/security
# Example backup command for /etc/gitlab:
# Create a time-stamped .tar file in the current directory.
# The .tar file will be readable only to root.
sudo sh -c 'umask 0077; tar -cf $(date "+etc-ssh-%s.tar") -C / etc/ssh'
```

If you ever have to restore, be mindful of permissions and ownership.
Reference: https://superuser.com/a/532079

**3. Application Data Backup**   

Reference: https://docs.gitlab.com/ce/raketasks/backup_restore.html#create-a-backup-of-the-gitlab-system  

We will let gitlab manage backups (See documentation if you want to manage it yourself). Also, we will backup to a locally mounted share. Also we will include all directories by default without skipping.

Reference: https://docs.gitlab.com/ce/raketasks/backup_restore.html#uploading-to-locally-mounted-shares

Before that let's mount the application data share and set permissions. The mounted share must be owned by the user `git`.

* By default, backup create will store a tar file in `/var/opt/gitlab/backups`. But we want to store your GitLab backups in a different directory, add the relevant setting to /etc/gitlab/gitlab.rb.
* The `backup_upload_remote_directory` must be set in addition to the `local_root` key. This is the sub directory inside the mounted directory that backups will be copied to, and will be created if it does not exist. If the directory that you want to copy the tarballs to is the root of your mounted directory, just use `.` instead.
* Set a limited lifetime for backups to prevent regular backups using all your disk space.

To configure gitlab to do the above, add the following settings to your existing gitlab.rb:
```
# Customization in /etc/gitlab/gitlab.rb, for backups omnibus installation
gitlab_rails['backup_path'] = '/home/gitadmin/gitlab/backups'

gitlab_rails['backup_upload_connection'] = {
  :provider => 'Local',
  :local_root => '/home/gitadmin/gitlab/backups'
}

# The directory inside the mounted folder to copy backups to
# Use '.' to store them in the root directory
gitlab_rails['backup_upload_remote_directory'] = 'application'

# Makes the backup archives world-readable
# Default permissions if not set: 0600
gitlab_rails['backup_archive_permissions'] = 0644

# limit backup lifetime to 7 days - 604800 seconds
# The backup_keep_time configuration option only manages local files.
gitlab_rails['backup_keep_time'] = 604800

```

Don't forget the reconfigure gitlab_rails:
```bash
sudo gitlab-ctl reconfigure
```

There are multiple back strategies. We will use copy (instead of tar or gzip). This will take additional 2x space as a side effect, but is safer.

To schedule a cron job that backs up your repositories and GitLab metadata, use the root user:
```bash
sudo su -
crontab -e
```
There, add the following line to schedule the backup for everyday at 2 AM.
```
0 2 * * * /opt/gitlab/bin/gitlab-rake gitlab:backup:create STRATEGY=copy CRON=1
```

if you ever have to run backup manually, below is the rake task command:
```bash
sudo gitlab-rake gitlab:backup:create STRATEGY=copy
```

**Restoring Application Backup**

First make sure your backup tar file is in the backup directory described in the gitlab.rb configuration gitlab_rails['backup_path']. The default is `/var/opt/gitlab/backups`.

```bash
sudo cp 1393513186_2014_02_27_gitlab_backup.tar /var/opt/gitlab/backups/
```

Stop the processes that are connected to the database. Leave the rest of GitLab running:
```bash
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq
# Verify
sudo gitlab-ctl status
```
Next, restore the backup, specifying the timestamp of the backup you wish to restore:
```bash
# This command will overwrite the contents of your GitLab database!
sudo gitlab-rake gitlab:backup:restore BACKUP=1393513186_2014_02_27
```

Restart and check GitLab:
```bash
sudo gitlab-ctl start
sudo gitlab-rake gitlab:check SANITIZE=true
```
