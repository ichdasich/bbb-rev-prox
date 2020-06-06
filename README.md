This documentation allows you to set up BBB behind a reverse proxy, for
example, if you only have one single IPv4 address, and a webserver that serves
other content is already running.

# System Basics

## Hosts and network configuration

This how-to uses five hosts:

- nat.home.aperture-labs.org [10.23.42.1 / 195.191.197.195] (Router/Default GW, OpenBSD 6.7)
- client1.home.aperture-labs.org [10.23.42.2] (Random client inside the network, Ubuntu 18.04)
- turn.home.aperture-labs.org [10.23.42.3] (TURN server for BBB, Ubuntu 18.04)
- web.home.aperture-labs.org [10.23.42.4] (Webserver and reverse proxy, Ubuntu 18.04)
- bbb.home.aperture-labs.org [10.23.42.5] (BBB Server, Ubuntu 16.04)

## DNS Configuration
There are two DNS records configured in the zone nat.home.aperture-labs.org

`nat.home.aperture-labs.org IN A 195.191.197.195`  
`*.nat.home.aperture-labs.org IN A 195.191.197.195`

## NAT/Routing configuration

On the gateway, IP forwarding is enabled, and the following NAT/Port-Forwarding
rules set. In case you do not use an OpenBSD router, please find the equivalent
settings for the network device you are using. Pull requests for
linux/cisco/etc. routers are welcome!

`
match out on hvn0 from 10.23.42.0/24 to any nat-to 195.191.197.195  
pass out on hvn0 from 10.23.42.0/24 to any  
  
; Redirect web-traffic to the web-host  
match in on hvn0 proto tcp from any to 195.191.197.195 port 80 rdr-to 10.23.42.4 port 80  
match in on hvn0 proto tcp from any to 195.191.197.195 port 443 rdr-to 10.23.42.4 port 443  
  
; Redirect TURN traffic on port 8443 to the turn host  
match in on hvn0 proto tcp from any to 195.191.197.195 port 8443 rdr-to 10.23.42.3 port 8443  
match in on hvn0 proto udp from any to 195.191.197.195 port 8443 rdr-to 10.23.42.3 port 8443  
  
; NAT internal hosts connecting to the external IP address of TURN/BBB, so they can also  
; use the external address/default DNS names
match out on hvn2 from 10.23.42.0/24 to 10.23.42.4 nat-to 10.23.42.1  
match out on hvn2 from 10.23.42.0/24 to 10.23.42.3 nat-to 10.23.42.1  
  
; Redirect internal traffic to the web-host (also necessary so that bbb api connects work  
; internally!  
match in on hvn2 proto tcp from any to 195.191.197.195 port 80 rdr-to 10.23.42.4 port 80  
match in on hvn2 proto tcp from any to 195.191.197.195 port 443 rdr-to 10.23.42.4 port 443  
  
; Same, but for TURN  
match in on hvn2 proto tcp from any to 195.191.197.195 port 8443 rdr-to 10.23.42.3 port 8443  
`

# Installing BBB

## 1. Add entries to /etc/hosts

We first have to add entries to /etc/hosts, so the servers can execute direct
connections when possible (and auto-configure to the right settings).
Furthermore, esp. on the web-host, it is necessary so that the proxy statements
via https work without certificate errors.

### web.nat.home.aperture-labs.org
`10.23.42.3 turn.nat.home.aperture-labs.org`  
`10.23.42.5 bbb.nat.home.aperture-labs.org`

### turn.nat.home.aperture-labs.org
`10.23.42.3 turn.nat.home.aperture-labs.org`

### bbb.nat.home.aperture-labs.org
`10.23.42.5      bbb.nat.home.aperture-labs.org`

## 2. Setup HTTP/80 Reverse Proxies

In the second step, we have to set up reverse proxies for TURN and BBB, so they
can get LE certificates automatically while using install.sh.

We add, to /etc/nginx/sites-enabled/default:

`
server  
{  
   listen 80;  
   listen       [::]:80;  
  
   server_name turn.nat.home.aperture-labs.org;  
  
   root /var/www/html;  
   location / {  
      proxy_pass http://turn.nat.home.aperture-labs.org;  
   }  
}  
  
  
  
server   
{  
   listen 80;  
   listen [::]:80;  
  
   server_name bbb.nat.home.aperture-labs.org;  
  
   root /var/www/html;  
   location / {  
      proxy_pass http://bbb.nat.home.aperture-labs.org;  
   }  
}  
`

# 3. Install TURN server
Now we can install and configure the TURN server on the turn host. 
  
`turn # wget -qO- https://ubuntu.bigbluebutton.org/bbb-install.sh | bash -s -- -c turn.nat.home.aperture-labs.org:use-another-secret -e your@mail.com`

## Configure TURN to use port 8443
After the script has run through, we can configure the TURN server. In 
`/etc/turnserver.conf` change:

-`tls-listening-port=443` to `tls-listening-port=8443`
-`external-ip=10.23.42.3` to `external-ip=195.191.197.195` (i.e. your external IP!)

Afterwards, restart the TURN server with `service coturn restart`.

# 4. Install BigBlueButton

Next, we can install BBB. As the install script does not like the difference 
between the local and remote IP address, we have to patch it. 

## Patching bbb-install.sh
Download the script with:  
`wget https://ubuntu.bigbluebutton.org/bbb-install.sh`
  
Then change lines 433 and 518 as follows:  
`
433c433  
<     local external_ip=$(grep $1 /etc/hosts| grep -o '^[.0-9]*'| tail -n1)  
---  
>     local external_ip=$(dig +short $1 @resolver1.opendns.com | grep '^[.0-9]*$' | tail -n1)  
518c518  
<     DIG_IP=$(grep $1 /etc/hosts| grep -o '^[.0-9]*'| tail -n1)  
---  
>     DIG_IP=$(dig +short $1 | grep '^[.0-9]*$' | tail -n1)  
`

## Running bbb-install.sh
Next, we can run the patched bbb-install.sh as usual:  
`cat bbb-install.sh | bash -s -- -v xenial-22 -s bbb.nat.home.aperture-labs.org -e your@mail.com -g -c turn.nat.home.aperture-labs.org:use-another-secret`

## Setting up syncing of https certificates

With this setup, the SSL certificates are generated (and automatically
refreshed) on bbb.nat.home.aperture-labs.org. Hence, we have to sync them over
to the web host, so they are also auto-generated there. Please figure out your
own method there.  For the test-setup, I opted for generating an ssh-key for
root on web.nat.home.aperture-labs.org and adding the pub-key on
bbb.nat.home.aperture-labs.org. I then setup the following cron-job ob
web.nat.home.aperture-labs.org:  

`23 1   *   *   *     rsync -av root@10.23.42.5:/etc/letsencrypt /etc && service nginx reload`

Note, that this might bring conflicts, if you are already using LE on web.nat.home.aperture-labs.org.

## Configuring TURN server

Next, change `/usr/share/bbb-web/WEB-INF/classes/spring/turn-stun-servers.xml`
to reflect the different TURN server port we are using:  

`sed -i s/443/8443/ /usr/share/bbb-web/WEB-INF/classes/spring/turn-stun-servers.xml`  

Apply this change with `bbb-conf --restart`; Depending on your setup, you might
want to add this change to the apply-config.sh script (see the BBB
documentation).

# 5. Configuring NGINX

After(!) you synced the SSL certificates, we can change the configuration on
web.nat.home.aperture-labs.org again, to add required proxy statements for
https/bbb. Change the vhost for bbb.nat.home.aperture-labs.org listening on
port 80 as follows:  

`
server  
{  
   listen 80;  
   listen [::]:80;  
    
   server_name bbb.nat.home.aperture-labs.org;  
    
   root /var/www/html;  
   index index.html;  
    
   location / {  
      proxy_pass http://bbb.nat.home.aperture-labs.org;  
   }  
   location /ws {  
      proxy_pass https://bbb.nat.home.aperture-labs.org:7443;  
      proxy_http_version 1.1;  
      proxy_set_header Upgrade $http_upgrade;  
      proxy_set_header Connection "Upgrade";  
      proxy_read_timeout 6h;  
      proxy_send_timeout 6h;  
      client_body_timeout 6h;  
      send_timeout 6h;  
   }  
   location /bbb-webrtc-sfu {  
      proxy_pass https://bbb.nat.home.aperture-labs.org;  
      proxy_http_version 1.1;  
      proxy_set_header Upgrade $http_upgrade;  
      proxy_set_header Connection "Upgrade";  
      proxy_read_timeout 6h;  
      proxy_send_timeout 6h;  
      client_body_timeout 6h;  
      send_timeout 6h;  
   }  
   location /pad/socket.io {  
      proxy_pass https://bbb.nat.home.aperture-labs.org;  
      proxy_set_header Host $host;  
      proxy_buffering off;  
      proxy_set_header X-Real-IP $remote_addr;  
      proxy_set_header X-Forwarded-For $remote_addr;  
      proxy_set_header X-Forwarded-Proto $scheme;  
      proxy_set_header Host $host;  
      proxy_http_version 1.1;  
      proxy_set_header Upgrade $http_upgrade;  
      proxy_set_header Connection "Upgrade";  
   }  
  
}  
`

And add the following new entry for https:  

`
server  
{  
   listen 443;  
   listen       [::]:443;  
   server_name bbb.nat.home.aperture-labs.org;  
   root /var/www/html;  
   ssl on;  
   ssl_certificate /etc/letsencrypt/live/bbb.nat.home.aperture-labs.org/fullchain.pem;  
   ssl_certificate_key /etc/letsencrypt/live/bbb.nat.home.aperture-labs.org/privkey.pem;  
   location / {  
      proxy_pass https://bbb.nat.home.aperture-labs.org;  
   }  
   location /ws {  
      proxy_pass https://bbb.nat.home.aperture-labs.org:7443;  
      proxy_http_version 1.1;  
      proxy_set_header Upgrade $http_upgrade;  
      proxy_set_header Connection "Upgrade";  
      proxy_read_timeout 6h;  
      proxy_send_timeout 6h;  
      client_body_timeout 6h;  
      send_timeout 6h;  
   }  
   location /bbb-webrtc-sfu {  
      proxy_pass https://bbb.nat.home.aperture-labs.org;  
      proxy_http_version 1.1;  
      proxy_set_header Upgrade $http_upgrade;  
      proxy_set_header Connection "Upgrade";  
      proxy_read_timeout 6h;  
      proxy_send_timeout 6h;  
      client_body_timeout 6h;  
      send_timeout 6h;  
   }  
   location /pad/socket.io {  
      proxy_pass https://bbb.nat.home.aperture-labs.org;  
      proxy_set_header Host $host;  
      proxy_buffering off;  
      proxy_set_header X-Real-IP $remote_addr;  
      proxy_set_header X-Forwarded-For $remote_addr;  
      proxy_set_header X-Forwarded-Proto $scheme;  
      proxy_set_header Host $host;  
      proxy_http_version 1.1;  
      proxy_set_header Upgrade $http_upgrade;  
      proxy_set_header Connection "Upgrade";  
   }  
}  
`

Afterwards, restart nginx with `service nginx restart`, and you are done. You
should now be able to reach your BBB server with enabled greenlight frontend at
https://bbb.nat.home.aperture-labs.org/b (if you installed it), and the BBB
server directly at https://bbb.nat.home.aperture-labs.org/. 

# 6. TODO

Currently, this needs a dedicated port-forward for TURN. I will do some more
digging to see if TURN can be reverse proxied as well (maybe with the WebSocket
Proxy options; Will update the documentation accordingly then.
