# **Apache HTTP Server Reverse Proxy**
We need to set up apache as a reverse-proxy on the host.

Download Apache 2.4.25 Win64 from here:
https://www.apachelounge.com/download/VC14/binaries/httpd-2.4.25-win64-VC14.zip

Unzip and install as a service using command window:
```
httpd.exe -k install
# if you have to uninstall:
# httpd.exe -k uninstall
```

Create a local user account and use that to start this service.

## Basic Settings to Test
Edit the httpd.conf with the following entries (lines 37, 246, 247):

```
ServerRoot "C:\Software\httpd-2.4.25-win64-VC14\Apache24"

DocumentRoot "C:\Software\httpd-2.4.25-win64-VC14\Apache24\htdocs"
<Directory "C:\Software\httpd-2.4.25-win64-VC14\Apache24\htdocs">
```
localhost:80 should say "It works!"

## Configure virtual hosts

Enable the following in httpd.conf
```
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

Comment the above three lines from httpd.conf (basic settings to test) and add the following:

```
<VirtualHost *:80>
    # This first-listed virtual host is also the default for *:80
    ServerName gitlab.localhost
    ServerAlias gitlab.localhost
    # Assuming that gitlab is serving from this port 9380 (forwarded to 80 on the virtualbox guest vm)
    ProxyPass / http://127.0.0.1:9380/
    ProxyPassReverse / http://127.0.0.1:9380/
    <Directory />
       AllowOverride none
       Require all denied
   </Directory>
</VirtualHost>
```

If you need to configure multiple virtual host see the reference bellow for name based virtual hosts:
Reference: https://httpd.apache.org/docs/2.4/vhosts/name-based.html  

## Configure HTTPS
We are going to configure self-signed certs on the front and apache httd server (used as a reverse proxy). Note that the nginx that serves gitlab on the virtualbox guest vm is using http (non-ssl). There is no need to enable ssl twice.

### Generate and Sign Certs
In command window:
```
set OPENSSL_CONF=C:\Software\httpd-2.4.25-win64-VC14\Apache24\conf\openssl.cnf
cd C:\Software\httpd-2.4.25-win64-VC14\Apache24\bin
openssl genrsa -des3 -out c:/Gitlab/ssl/gitlab.local.key 4096
openssl req -new -key c:/Gitlab/ssl/gitlab.local.key -out c:/Gitlab/ssl/gitlab.local.csr
```

In git bash
```bash
cp -v c:/Gitlab/ssl/gitlab.local.{key,original}
```

In command window:
```
openssl rsa -in c:/Gitlab/ssl/gitlab.local.original -out c:/Gitlab/ssl/gitlab.local.key
# try in bash if this doesnt work in command
rm -v c:/Gitlab/ssl/gitlab.local.original
openssl x509 -req -days 1460 -in c:/Gitlab/ssl/gitlab.local.csr -signkey c:/Gitlab/ssl/gitlab.local.key -out c:/Gitlab/ssl/gitlab.local.crt
```

### Test Basic HTTPS

We are enabling SSLSessionCache. We will listen to both http and https requests and forward http to https.

In httpd.conf
```
Listen 80
Listen 443

LoadModule ssl_module modules/mod_ssl.so

SSLSessionCache        "shmcb:C:\Software\httpd-2.4.25-win64-VC14\Apache24/logs/ssl_scache(512000)"
SSLSessionCacheTimeout  300

Include conf/extra/httpd-vhosts.conf
```
In httpd-vhosts.conf
```
<VirtualHost *:80>
   ServerName gitlab.local
   Redirect permanent / https://gitlab.local/
</VirtualHost>

<VirtualHost _default_:443>
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl on
    RequestHeader set X-URL-SCHEME https
    ServerName gitlab.local:443
    ServerAlias gitlab:443
    ErrorLog "C:\Software\httpd-2.4.25-win64-VC14\Apache24/logs/gitlab-error.log"
    CustomLog "C:\Software\httpd-2.4.25-win64-VC14\Apache24/logs/gitlab-access.log" common
    # ServerAdmin admin@example.com
    DocumentRoot "C:\Software\httpd-2.4.25-win64-VC14\Apache24\htdocs"
    <Directory />
       Require all granted
   </Directory>


    SSLEngine On
    SSLCertificateFile c:/Gitlab/ssl/gitlab.local.crt
    SSLCertificateKeyFile c:/Gitlab/ssl/gitlab.local.key
    <Location />
       SSLRequireSSL
    </Location>
</VirtualHost>
```

Verify both https and http redirect

### Set up a reverse-proxy to gitlab using https

**Note:** NGINX on the virtualbox guest vm needs to be configured when the reverse-proxy is set this way. Please refer to NGINX.md to reconfigure it.

Reference: https://wiki.apache.org/httpd/RedirectSSL

Make sure mod_ssl, mod_proxy & mod_proxy_http are enabled (see above steps). Additionally enable mod_headers:
In httpd.conf, uncomment the following line:
```
LoadModule headers_module modules/mod_headers.so
```

Add the following in the extras/httpd-vhosts.conf. Make sure the entries for SSLCertificateFile and SSLCertificateKeyFile point to the openssl certificate files you created above.  e.g.:

Also, note that the server name has to match

```
<VirtualHost *:80>
   ServerName gitlab.local
   Redirect permanent / https://gitlab.local/
</VirtualHost>

<VirtualHost _default_:443>
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl on
    RequestHeader set X-URL-SCHEME https
    ServerName gitlab.local:443
    ServerAlias gitlab:443
    ErrorLog "C:\Software\httpd-2.4.25-win64-VC14\Apache24/logs/gitlab-error.log"
    CustomLog "C:\Software\httpd-2.4.25-win64-VC14\Apache24/logs/gitlab-access.log" common
    # ServerAdmin admin@example.com

    SSLEngine On
    SSLCertificateFile c:/Gitlab/ssl/gitlab.local.crt
    SSLCertificateKeyFile c:/Gitlab/ssl/gitlab.local.key
    <Location />
       SSLRequireSSL
    </Location>

    ProxyPreserveHost On
    ProxyPass / http://gitlab.local:9380/
    ProxyPassReverse / http://gitlab.local:9380/
</VirtualHost>
```

Verify that both https and http redirect work. Both should be proxied to gitlab.

### Log rotation
Reference: https://httpd.apache.org/docs/2.4/programs/rotatelogs.html  
We will configure apache to rotate every 24 hrs and clean files older than 7 days. Add the following to httpd-vhosts.conf
```
<VirtualHost *:80>
   ServerName gitlab.local
   Redirect permanent / https://gitlab.local/
</VirtualHost>

<VirtualHost _default_:443>
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Ssl on
    RequestHeader set X-URL-SCHEME https
    ServerName gitlab.local:443
    ServerAlias gitlab.local:443
    ErrorLog "|C:/alohaqs/Software/httpd-2.4.25-win64-VC14/Apache24/bin/rotatelogs.exe -l C:/alohaqs/Software/httpd-2.4.25-win64-VC14/Apache24/logs/gitlab-error.%Y-%m-%d.log 86400"
    CustomLog "|C:/alohaqs/Software/httpd-2.4.25-win64-VC14/Apache24/bin/rotatelogs.exe -l C:/alohaqs/Software/httpd-2.4.25-win64-VC14/Apache24/logs/gitlab-access.%Y-%m-%d.log 86400" combinedtrueout_host
    CustomLog "|C:/alohaqs/Software/httpd-2.4.25-win64-VC14/Apache24/bin/rotatelogs.exe -l C:/alohaqs/Software/httpd-2.4.25-win64-VC14/Apache24/logs/gitlab-deflate.%Y-%m-%d.log 86400" deflate
    # ServerAdmin admin@example.com

    SSLEngine On
    SSLCertificateFile c:/Gitlab/ssl/gitlab.local.crt
    SSLCertificateKeyFile c:/Gitlab/ssl/gitlab.local.key
    <Location />
       SSLRequireSSL
    </Location>

    ProxyPreserveHost On
    ProxyPass / http://gitlab.local:9380/
    ProxyPassReverse / http://gitlab.local:9380/
</VirtualHost>
```

### Configure trusted_proxies and real_ip:

### Hardening Apache SSL Configuration:
TODO
