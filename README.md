# ssl_domoticz
ssl domoticz


Native secure access with Lets Encrypt
Jump to navigationJump to search
Domoticz includes a SSL certificate generated for the *.domoticz.com domain. So for your own domain, it may result some warning error by your browser.

This article shows how to add a LetsEncrypt certificate to Domoticz so you can access your server over a secure HTTPS channel and without warning error. This certificate has a lifetime of 3 months and must be renewed every 3 months.

The provided steps are executed using a Raspberry Pi, but they should work on every Linux OS.

Prerequisites (see here : http://www.domoticz.com/wiki/Native_HTTPS_/_SSL_support)

You own a domain name
The (sub)domain name for Domoticz has a DNS entry that points to your external IP address
Port 80 (HTTP) and 443 (HTTPS) , in your internet box, are forwarded to your Domoticz server. Forwarding Port 80 is only needed for the creation and renewal time. I advice that you disable it after.
domoticz must listen to the port HTTP and HTTPS. Check this in the startup script : /etc/init.d/domoticz.sh


Contents
1	Install Lets Encrypt
2	Create the certificate
3	Add the certificate to Domoticz
4	Renew the certificate automatically
Install Lets Encrypt
Note that the current Lets Encrypt procedures refer to certbot-auto (October 2019)
```
cd /etc
sudo git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
sudo ./letsencrypt-auto
```
On Pi this last command will take a long time ...

if it fails due to a hash error, the culprit is this file (in july 2019) which points at a repo which is not sound see:
```
sudo nano /etc/pip.conf
```
Move it and reboot... and run the last command again:
```
sudo mv /etc/pip.conf /etc/pip.conf.backup 
sudo reboot
sudo ./letsencrypt-auto
```
will fix it.

Create the certificate
```
sudo /etc/letsencrypt/letsencrypt-auto certonly --webroot --email <your email> -d <your complete sub.domain name> -w <user home>/domoticz/www/
````
(check that your domoticz is accessible on the port HTTP 80 via NAT forwarding in your router)

Letsencrypt create a temporarly file in the www directory of domoticz. This file will be checked by the letsencrypt server to ensure that you are the owner of the domain. Then it remove the temporarly file.


Edit Sep 10 2017 : If you do not want to expose port HTTP 80 to the outside world you can also use --preferred-challenges=dns and create a DNS TXT record (as described) to validate the ownership


You can specify multiple domain names using another -d parameter and domain name for each additional domain name.


If everything is OK this message shows:

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/<your domain>/fullchain.pem. Your
   cert will expire on <date>. To obtain a new version of the
   certificate in the future, simply run Let's Encrypt again.
 - If you like Let's Encrypt, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
The certificate is created in /etc/letsencrypt/live/

Add the certificate to Domoticz
Then you add the created certificate to Domoticz with :
```
sudo mv ~/domoticz/server_cert.pem ~/domoticz/server_cert.pem.org  # see below about DH params why not just delete it
sudo cat /etc/letsencrypt/live/<your domain>/privkey.pem >> ~/domoticz/server_cert.pem
sudo cat /etc/letsencrypt/live/<your domain>/fullchain.pem >> ~/domoticz/server_cert.pem
sudo cp ~/domoticz/server_cert.pem ~/domoticz/letsencrypt_server_cert.pem
sudo /etc/init.d/domoticz.sh restart
```
As every update of domoticz overwrites your certificate, the last command backups your new certificate so that you may may restore it if needed.

When there's a domoticz error after rebooting the service like : Error: [web:443] missing SSL DH parameters from file

Add the DHparam :
```
sudo cat /etc/ssl/certs/dhparam.pem >> ~/domoticz/server_cert.pem
```
and if you get also an error like : /etc/ssl/certs/dhparam.pem: No such file or directory
```
cd /etc/ssl/certs
sudo openssl dhparam -out dhparam.pem 2048
sudo cat /etc/ssl/certs/dhparam.pem >> ~/domoticz/server_cert.pem
sudo /etc/init.d/domoticz.sh restart
```
Here, the openssl command will take a very long time (like 45 minutes on a Pi3). Alternatively you could copy the section from the original server_cert.pem file:

sudo tail -n 8 server_cert.pem.org >> ~/domoticz/server_cert.pem
This will append the lines from -----BEGIN DH PARAMETERS----- to -----END DH PARAMETERS----- to the end of the pem file.

Renew the certificate automatically
Your Lets Encrypt certificate is valid for 3 months. To automtically renew it, you could make a cron job to try and renew it say every week. To do so:
```
crontab -e
```
and add a line like

0 23 * * 5 sudo certbot-auto renew --webroot -w ~/domoticz/www/ --deploy-hook ~/domoticz/scripts/deploy-cert.sh >/dev/null

Note that I use certbot-auto here instead of letsencrypt-auto because I know the above works. To install the renewed certificate --deploy-hook is used with a script to create the new server_cert.pem with correct contents as described above.
