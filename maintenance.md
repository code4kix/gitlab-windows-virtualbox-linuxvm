# ** Gitlab Omnibus - Maintenance Documentation **

## Upgrading Gitlab

**_IMPORTANT_** UPDATE sandbox before the real one Reference: <https://about.gitlab.com/upgrade-to-package-repository/#centos>

ssh into the virtualbox guest vm and run the following in bash shell terminal:

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo yum install gitlab-ce
```
