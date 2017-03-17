# **Gitlab Omnibus - LDAP Configuration**

We will create a file: `ldap_settings.yml` file in the host system and scp it onto the virtualbox guest vm. After this we will edit the local copy of gitlab.rb to use this yml file and scp it too. Note that the indentation matters in the yml file.

Reference: <https://www.bonusbits.com/wiki/HowTo:Setup_LDAP_on_Gitlab><br>
The contents of `ldap_settings.yml` are as like so
```
main:
  host: 'fin.ds.ky.gov'
  port: 389
  uid: 'sAMAccountName'
  method: 'tls' # 'tls' or 'plain' or 'ssl'
  bind_dn: '_the_full_dn_of_the_user_you_will_bind_with'
  password: '_the_password_of_the_bind_use'
  active_directory: true
  # If you are using "uid: 'userPrincipalName'" on ActiveDirectory you need to
  # disable this setting, because the userPrincipalName contains an '@'.
  allow_username_or_email_login: false
  base: 'ou=People,dc=gitlab,dc=example'
  user_filter:
  attributes:
      username: [ 'sAMAccountName' ]
      email: [ 'mail', 'userPrincipalName' ]
      name: [ 'displayName' ]
```

Now let's copy it to ~ on the virtualbox guest vm (you won't have permission to directly copy it to `` `/etc/gitlab/``). Open bash terminal on host:

```bash
# copy it to ~ (you won't have permission to directly copy it to /etc/gitlab/)
scp -p 2223 /c/Gitlab/misc/ldap_settings.yml gitadmin@localhost:
# then copy it to /etc/gitlab/
sudo cp ldap_settings.yml /etc/gitlab/ldap_settings.yml
```

Now ssh into the virtualbox guest vm:

```bash
# Optionally backup existing gitlab.rb template (for reference) and then replace it
sudo cp /etc/gitlab/gitlab.rb /etc/gitlab/gitlab.backup.rb
sudo cp ~/gitlab.rb /etc/gitlab/gitlab.rb
```

Let's add the following to `gitlab.rb`
```
gitlab_rails['ldap_enabled'] = true
gitlab_rails['ldap_servers'] = YAML.load_file('/etc/gitlab/ldap_settings.yml')

# turn off sign-up after creating the admin.
# this config setting below doesn't work anymore. So do it from the gitlab web ui: admin > settings
# gitlab_rails['gitlab_signup_enabled'] = false
```
Reconfigure and test
```
sudo gitlab-ctl reconfigure
```
