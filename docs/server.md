# Server Documentation

This section is to catalog server configurations that I've built for future use.

## Minecraft Forge Server Setup
This walkthrough goes through how to setup a Minecraft Forge server. This should work with your own hardware or on hardware in a cloud. If you need to use hardware in a cloud, I recommend Linode.

### Install the Operating System
To get started download the Linux operating system that you want to use for this setup. I used Ubuntu 22.04 LTS and the installation worked fine. Go through the normal installation steps for the server. I recommend setting a static IP address on the server if you are working with your own hardware.

Once the server is installed, make sure the following applications are installed and configured:

1. SSH (`sudo apt install ssh`). Check this [other article](https://www.spoonnet.net/wiki/system/#ssh-configuration) for how to setup SSH in the most secure way.

2. Tailscale - go [here](https://tailscale.com/download/linux) and run the script at the top of the page. Follow the steps to get it installed. Tailscale is a wireguard enabled system to system connectivity application similar to VPN.

3. Install java. For most all newer Forge servers you will need Java OpenJDK 17. The command for that is `sudo apt install openjdk-17-jdk`

Then, configure the server for a specific minecraft user. It's best to keep the installation under a specific user for security and separation of responsibilities. To do this check out this [other article](https://www.spoonnet.net/wiki/system/#creating-a-new-user-with-super-user-privileges).

### Download the Minecraft Forge server
The next step is to download the Minecraft Forge server. You can find it [here](https://files.minecraftforge.net/net/minecraftforge/forge/). Once you've downloaded it, get it onto your server via SSH with the minecraft user.

Once you are SSH'd in, create a new directory under your home path `mkdir ~/minecraft`. Copy the forge installation file into this folder with SCP. You can use command-line SCP in Linux or WinSCP in Windows.

You then run this command to install the server `java -jar forge-x.xx.x-installer.jar --installServer`. Change out the 'X's to the version that you downloaded.

The first time you run this command it will stop and say that you must accept the EULA. A EULA.txt file will be created. Open this file with nano and change the value to true. Close it out and run the installation Jar again.

This should install the forge server and will create a run.sh file. To start the server, you can just run this file by typing `./run.sh`.

### Uploading mods
The purpose of Minecraft Forge is to be able to install mods on the server. Keep in mine, you have to have the same version of the mod on the PC you will use to play Minecraft (as well as the same version as forge). 

There are multiple ways to get the mods into the $HOME/minecraft/minecraft/mods folder. You can check out this article to see a way to do this using syncthing or you can just SCP these files to the server each time.

Keep in mind as well that you will have to stop the server with `/stop` and then start the server again with `./run.sh` after adding the mods in the mods folder for them to take effect.

Also a note, I run the `./run.sh` command inside of a Linux screen. This ensures I can keep it running in the background and can visit it if anything crashes to see the latest logs.

### Contact
For any questions or updates that you have for this material, please reach out to me through [Discord](https://discord.gg/RBBHZ5b3AE)

## NGINX Hosting Server with Docker
This documentation attempts to outline my implementation of NGINX and using NGINX as a proxy for docker containers. The main purpose of implementing NGINX in this way is to only keep up with one SSL certificate and also to use one website and TCP ports (80,443) to host multiple web applications. This is helpful when you are self hosting servers and have limited IP address space.

### Installing NGINX
Like other posts this is specific to Ubuntu server specifically 22.04. Other distros implementation should be pretty similar though. To install NGINX issue the command `sudo apt install nginx`. Once this is installed you should be able to navigate in a browser to http://127.0.0.1 and see the normal default webpage. If you do not see this webpage issue the `sudo service nginx status` command to see if the service is started. If it's not started issue the `sudo service nginx start` command to start it up.

### Let's Encrypt SSL Certificate
Next, I use Let's Encrypt to generate a free 3-month SSL certificate to use on my personal projects. To do this on the server you must first purchase a domain and point the URL you want to use to the public IP that the server is using. You must also confirm that your upstream router/firewall/modem has a port forward for the port you are using, specifically ports 80 and 443.

Once you have your DNS and port forwarding setup, install certbot on the server by issuing the command `sudo apt install certbot`. Once it is installed, run it in certonly mode. This will spin up a temporary server on port 80 from Let's Encrypt servers to ensure domain ownership and then will issue the key. The command to run this is `sudo certbot certonly`. Follow the prompts and choose the temporary server option. If NGINX is already started, you have to stop it first before running the certbot command. To stop the NGINX server issue the command `sudo service nginx stop`.

Keys are normally issued to /etc/letsencrypt/live/-domain name-/fullchain.pem and privkey.pem. You will need these paths when configuring NGINX.

### NGINX Configuration
I've given a sample configuration below for NGINX. This ties docker containers are even local services to specific paths on the web server. I've laid out each section below. To update yours you will need to edit the file /etc/nginx/sites-enabled/default (unless you have another configuration file).


    #This section configured aliases used further down in the config for the docker IP and the port that the service on docker is running on. Keep in mind this is not docker forwarded port but the actual port the service is running on in Docker.

    upstream SpoonWebsite {
        server 172.17.0.2:80;
    }
    upstream SpoonBlog {
        server 172.18.0.2:2368;
    }
    #This entry is an example of a service running on the actual system outside of a Docker container.
    upstream SpoonWiki {
        server localhost:8000;
    }

    #This section sets up what happens when port 80 is visited for the website. Below, I have it setup to just forward to HTTPS.
    server {
        listen 80;
        listen [::]:80;
        server_name spoonnet.net www.spoonnet.net;

        location / {
            rewrite ^ https://$host$request_uri? permanent;
        }
        if ($host = www.spoonnet.net) {
            return 301 https://$host$request_uri;
        } # managed by Certbot


        if ($host = spoonnet.net) {
            return 301 https://$host$request_uri;
        } # managed by Certbot

            server_name spoonnet.net www.spoonnet.net;
        return 404; # managed by Certbot

    }

    #This is the most important section. Here you will need to update the ssl_certificate lines
    server {
            server_name spoonnet.net www.spoonnet.net;

        listen [::]:443 ssl ipv6only=on; # managed by Certbot
        listen 443 ssl; # managed by Certbot
        ssl_certificate /etc/letsencrypt/live/www.spoonnet.net/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/www.spoonnet.net/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

            #This is the default location. Note that for proxy_pass it points to the alias above for the Docker website container.
            location / {
                    proxy_pass http://SpoonWebsite;
                    proxy_redirect off;
                    proxy_set_header Host   $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header   X-Forwarded-Host $server_name;
            }
            #This is for subdirectory blog and points to a separate alias docker container.
            location /blog {
                    proxy_pass http://SpoonBlog;
                    proxy_redirect off;
                    proxy_set_header Host   $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header   X-Forwarded-Host $server_name;
            }
            #Same as blog above
            location /wiki {
                    proxy_pass http://SpoonWiki;
                    proxy_redirect off;
                    proxy_set_header Host   $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header   X-Forwarded-Host $server_name;
            }
    }

Once you have updated your configuration as needed, you will need to restart the NGINX service for the changes to take effect. Do this by issuing `sudo service nginx restart`.

### Setting up a Docker container
There are many ways to do this and many use cases. I will create a future Wiki to outline this more.