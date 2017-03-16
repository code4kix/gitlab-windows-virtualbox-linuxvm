# ** Gitlab Omnibus - NGINX Configuration **

Reference: https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/nginx.md

Let's create a local copy of `gitlab.rb` on the host and scp it onto the virtualbox guest vm.  

### Set up reverse-proxy on the host
Since the host is windows, let us use apache httpd server.  
See [apache config on host](apache.md)  
Note: No need to configure NGINX on the virtualbox guest vm for https; we will just use http.

### Proxied SSL

By default NGINX will auto-detect whether to use SSL if external_url
contains https://.  If you are running GitLab behind a reverse proxy, you
may wish to terminate SSL at another proxy server or load balancer. To do this,
be sure the external_url contains https:// and apply the following
configuration to gitlab.rb:
```
# note the 'https' below
external_url "https://gitlab.local"

nginx['listen_port'] = 80
nginx['listen_https'] = false
nginx['proxy_set_headers'] = {
  "X-Forwarded-Proto" => "https",
  "X-Forwarded-Ssl" => "on"
}
```

Note that you may need to configure your reverse proxy or load balancer to
forward certain headers (e.g. Host, X-Forwarded-Ssl, X-Forwarded-For,
X-Forwarded-Port) to GitLab. You may see improper redirections or errors
(e.g. "422 Unprocessable Entity", "Can't verify CSRF token authenticity") if
you forget this step. For more information, see:


http://stackoverflow.com/questions/16042647/whats-the-de-facto-standard-for-a-reverse-proxy-to-tell-the-backend-ssl-is-used
https://wiki.apache.org/couchdb/Nginx_As_a_Reverse_Proxy
### Configuring GitLab trusted_proxies and the NGINX real_ip module

By default, NGINX and GitLab will log the IP address of the connected client.

If your GitLab is behind a reverse proxy, you may not want the IP address of
the proxy to show up as the client address.

You can have NGINX look for a different address to use by adding your reverse
proxy to the real_ip_trusted_addresses list. Assuming 10.0.2.2 is the ip of your virtualbox guest vm, add the following config to etc/gitlab/gitlab.rb:
```
# Each address is added to the the NGINX config as 'set_real_ip_from <address>;'
nginx['real_ip_trusted_addresses'] = [ '10.0.2.2' ]
# other real_ip config options (X-Forwarded-For is set by apache httpd reverse-proxy)
nginx['real_ip_header'] = 'X-Forwarded-For'
nginx['real_ip_recursive'] = 'on'
```

Description of the options:
http://nginx.org/en/docs/http/ngx_http_realip_module.html


By default, omnibus-gitlab will use the IP addresses in real_ip_trusted_addresses
as GitLab's trusted proxies, which will keep users from being listed as signed
in from those IPs.

Save the file and reconfigure GitLab
for the changes to take effect.



### Appendix: NGINX - Enable SSL - Self-Signed Certs:

Note that this step is not needed if you use a reverse-proxy. But if you need to enable ssl using self-signed certs on nginx, follow along. You can use letsencrypt or rapidssl if you need to (just follow the above reference).  

Reference: <https://www.bonusbits.com/wiki/HowTo:Setup_HTTPS_for_Gitlab>

If you want to enable HTTPS for
gitlab.local, add the following statement to /etc/gitlab/gitlab.rb:
```bash
# enable https
# note the 'https' below
external_url "https://gitlab.local"
# nginx will no longer listen for unencrypted HTTP traffic on port 80, unless you have this line below
nginx['redirect_http_to_https'] = true
```

** Generate keys: ** On the virtualbox guest machine. Open a shell terminal and ssh into the virtualbox guest vm. Note here that the keys are named `gitlab.example.com` because that is used in the `external_url` entry in the gitlab.rb file. These key names below has to match that entry:

```bash
sudo openssl genrsa -des3 -out /etc/gitlab/ssl/gitlab.example.com.key 4096
# Asks for a passphrase
sudo openssl req -new -key /etc/gitlab/ssl/gitlab.example.com.key -out /etc/gitlab/ssl/gitlab.example.com.csr
# Important to use the FQDN for CN field. In my case I will use gitlab.example.com
# Remove Passphrase from Private Key
sudo cp -v /etc/gitlab/ssl/gitlab.example.com.{key,original}
sudo openssl rsa -in /etc/gitlab/ssl/gitlab.example.com.original -out /etc/gitlab/ssl/gitlab.example.com.key
sudo rm -v /etc/gitlab/ssl/gitlab.example.com.original
# create cert
sudo openssl x509 -req -days 1460 -in /etc/gitlab/ssl/gitlab.example.com.csr -signkey /etc/gitlab/ssl/gitlab.example.com.key -out /etc/gitlab/ssl/gitlab.example.com.crt
# remove csr
sudo rm -v /etc/gitlab/ssl/gitlab.example.com.csr
# set permissions
sudo chmod 600 /etc/gitlab/ssl/gitlab.example.com.*
```
Note that if you are using a firewall you may have to open port 443 to allow inbound
HTTPS traffic.

```bash
# firewall-cmd (RedHat, Centos 7)
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld
```

To set the location of ssl certificates create /etc/gitlab/ssl directory,
place the .crt and .key files in the directory and specify the following
configuration:
```bash
# For GitLab
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.example.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.example.com.key"
```

Run `sudo gitlab-ctl reconfigure` for the change to take effect.


# Optional - to monitor port and verify everything works install netstat on CentOS-7
sudo yum install net-tools
#Verify
sudo netstat -tan | grep 443
```
