
# Arimac Meet on Docker

![](resources/docker-jitsi-meet.png)

I have implemented **[Arimac Meet](https://meet.arimac.digital)** on Docker using Jitsi Open Source project.

[Jitsi](https://jitsi.org/) is a set of Open Source projects that allows you to easily build and deploy secure videoconferencing solutions.

Arimac Meet is a fully encrypted, 100% Open Source video conferencing solution that you can use all day, every day, for free — with no account needed.

This repository contains the necessary tools to run a Arimac Meet stack on [Docker](https://www.docker.com) using [Docker Compose](https://docs.docker.com/compose/).

## Prerequisites

Before setting up Arimac Meet on your server, make sure you have following software are installed on your host machine:

 - [Docker](https://docs.docker.com/engine/install/)
 - [Docker Compose](https://docs.docker.com/compose/install/)
 - [Nginx](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/) (Native) 
***Note:** you can skip Nginx native installation by using Nginx service on Docker as a service itself*

## Installation
Follow these basic instructions to install to customize Arimac Meet anywhere.

### Open Required Ports in Firewall
The following external ports must be opened on a firewall:
- 80/tcp for Web UI HTTP 
- 443/tcp for Web UI HTTPS
- 4443/tcp for RTP media over TCP
- 10000/udp for RTP media over UDP

- Also 20000-20050/udp for jigasi, in case you choose to deploy that to facilitate SIP access.

### Basic Setup & Configuration
You can basically do following common steps in order to bring Arimac Meet alive on your server:

**Step 1:** Clone this repo in to your server as follows:
```
git clone git@gitlab.com:mr_arimac/arimac-meet.git arimac_meet
```
**Step 2:** Copy `env.example` into `.env` :
```
cd arimac_meet
sudo cp env.example .env
```
Now, edit `.env` file as your preferences.

**Step 3:** Run  `gen-passwords.sh` script to generate required inter communication passwords:
```
sudo chmod +x gen-passwords.sh
sudo ./gen-passwords.sh
```
**Step 4:** Create required `CONFIG` directories:
```
sudo mkdir -p ~/.jitsi-meet-cfg/{web/letsencrypt,transcripts,prosody/config,prosody/prosody-plugins-custom,jicofo,jvb,jigasi,jibri}
```

**Step 5:** Customize frontend:
You can now inject your custom files to localhost mounted location.
```
cd arimac_meet/arimac-meet-ui/web-customization
```
You can add your customization here and exit. Then, all is ready.

**Step 6:** Set up Jibri for video recording:
Before running Jibri, you need to set up an ALSA loopback device on the host. This  **will not**  work on a non-Linux host.

For CentOS 7/8, the module is already compiled with the kernel, so just run:
```
# configure 5 capture/playback interfaces  
echo "options snd-aloop enable=1,1,1,1,1 index=0,1,2,3,4" > /etc/modprobe.d/alsa-loopback.conf # setup autoload the module  
echo "snd_aloop" > /etc/modules-load.d/snd_aloop.conf 
# load the module 
modprobe snd-aloop 
# check that the module is loaded 
lsmod | grep snd_aloop
```
For Ubuntu:
```
# install the module 
apt update && apt install linux-image-extra-virtual 
# configure 5 capture/playback interfaces  
echo  "options snd-aloop enable=1,1,1,1,1 index=0,1,2,3,4" > /etc/modprobe.d/alsa-loopback.conf
# setup autoload the module  
echo  "snd-aloop" >> /etc/modules 
# check that the module is loaded 
lsmod | grep snd_aloop
```
***NOTE:*** If you are running on AWS you may need to reboot your machine to use the generic kernel instead of the "aws" kernel. If after reboot, your machine is still using the "aws" kernel, you'll need to manually update the grub file. So just run:
```
# open the grub file in editor 
nano /etc/default/grub 
# Modify the value of GRUB_DEFAULT from "0" to "1>2"  
# Save and exit from file  

grub update-grub 
sudo reboot
```
**Step 7:** Set up Etherpad services:
If you like, additionally you can setup Etherpad for documentation purposes while on meeting.
In the `docker-compose.yml` file, I've already included etherpad services.

Only thing you have to do is defining etherpad domain and related stuff.

**Step 8:** Start application:
```
docker-compose up -d
```
Now all is set go!

*For advanced setup you can refer official Jitsi documentation [here](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker).*

Your **Arimac Meet** is now live on one of your domain as bellow example: 

 - http://meet.arimac.digital:8000 
 - https://meet.arimac.digital:8443

## Nginx Reverse Proxy Setup
We can use Nginx reverse proxy to handle the traffic into our application properly.

**Step 1:** create a configuration block:
```
vi /etc/nginx/conf.d/meet.arimac.digital.conf
```
Change your domain name accordingly.

Then, add following content:
Make sure to change, `server_name` value with your domain name.
```
server {
   server_name meet.arimac.digital;
   listen 80;
   
   location / {
	ssi on;
	proxy_pass https://localhost:8443;
	proxy_set_header X-Forwarded-For $remote_addr;
	proxy_set_header Host $http_host;

	}

   location /xmpp-websocket {
        proxy_pass https://localhost:8443/xmpp-websocket;
    	proxy_http_version 1.1;
    	proxy_set_header Upgrade $http_upgrade;
    	proxy_set_header Connection "upgrade";
	}

   location /colibri-ws/ {
    	proxy_pass https://localhost:8443/colibri-ws/;
    	proxy_http_version 1.1;
    	proxy_set_header Upgrade $http_upgrade;
    	proxy_set_header Connection "upgrade";
	}

   # Etherpad-lite
   location ^~ /etherpad/ {
       proxy_pass http://localhost:9001/;
       proxy_set_header X-Forwarded-For $remote_addr;
       proxy_buffering off;
       proxy_set_header       Host $host;
   }
}

server {
    if ($host = meet.arimac.digital) {
        return 301 https://$host$request_uri;
    } 

   server_name meet.arimac.digital;
   listen 80;
}
```
**Step 2:** Make sure Nginx configuration is resolving successfully:
```
sudo nginx -t
```
**Step 3:** Restart Nginx and make sure it's running properly:
```
sudo systemctl restart nginx
sudo systemctl status nginx
```

## Let's Encrypt Free SSL

Arimac Meet requires SSL to ensure all communications are done properly with the highest security it has.
In this case, we can provision a free SSL Certificate from Let's Encrypt SSL Authority. 

***Note:** Before going forward, please make sure Meet domain is correctly resolving to your host machine's public IP Address.*

**Step 1:** Install Certbot:

Please visit https://certbot.eff.org and select Nginx as the software and your system as preffered in order to install certbot manager on your host machine.
Now all is ready to provision a free auto-renewing SSL certificate for your Meet domain.

**Step 2:** Provision SSL Certificate

```
sudo certbot --nginx -d meet.arimac.digital -d www.meet.arimac.digital
```
Make sure to replace your domain name with `meet.arimac.digital` .


## TODO

* Setup CI/CD for interface customization (where applicable).
* Improve High Availability using HAProxy
* Using TURN server

## How to Contribute

If you spot any bugs, please use  [Gitlab Issues](https://gitlab.com/mr_arimac/arimac-meet/-/issues)  or if you want to add a new feature directly   [Fork](https://gitlab.com/mr_arimac/arimac-meet/-/forks/new) and push to the main repo.

## Sponsoring

If you like my work and want to support the development of the project, now you can! Just:

[![Buy Me A Coffee](https://camo.githubusercontent.com/0f3e240e3bc1e7876ec84f7d3e224eb01144293e5a74e7f53df732d656bfc325/68747470733a2f2f7265732e636c6f7564696e6172792e636f6d2f70616e722f696d6167652f75706c6f61642f76313537393337343730352f6275796d6561636f666665655f793679766f762e737667)](https://manoranjana.me)

## License

### Apache License 2.0

A permissive license whose main conditions require preservation of copyright and license notices. Contributors provide an express grant of patent rights. Licensed works, modifications, and larger works may be distributed under different terms and without source code.
*For more info. please read [here](https://www.apache.org/licenses/LICENSE-2.0).*