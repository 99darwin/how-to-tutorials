# How To Run a NodeJS Application with Cloudflare and Digital Ocean

This tutorial assumes that you already have a domain, a web app you'd like to host uploaded to GitHub, some basic knowledge of how to setup a Digital Ocean droplet, and are using Cloudflare DNS. 

**Example DNS settings:**
![example dns settings](/settings.png)

*Note: Your domain registrar will have instructions on how to point your DNS toward Cloudflare. The free Cloudflare subscription **will** work for this tutorial.*

In this tutorial we will be covering:
* Installing Node and NPM with NVM on a Linux Ubuntu server.
* Installing a NodeJS application on a Digital Ocean VPS
* Installing Nginx on a Digital Ocean VPS
* Acquiring an SSL certificate for your domain
* Creating a reverse proxy to your hosting port
* Enabling DNS routing through Cloudflare

# Step 1: Create a new Digital Ocean Droplet
Go to your Digital Ocean account and create a new droplet. Select your desired settings and be sure to add your SSH key. Name the server and click Create.

# Step 2: Create DNS records in Cloudflare
Under the DNS tab in Cloudflare, add two records and turn off Cloudflare. **To do this, click the cloud under Status so that it turns grey**. 

1. Create an A record with `yourdomain.com` as the name and your server's IP as the Value. 
2. Create a CNAME record with `www` as the name. This CNAME will be an alias of your domain.

Leave TTL as Automatic.

Again, it is very important that you deactivate Cloudflare at this point. Once again, click the cloud under Status so that it turns grey, we will come back to this later.

# Step 3: Install Node on your server
Access your server from a terminal window

`ssh root@yourserverip`

Once on your server

```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get install build-essential libssl-dev
```

Pull down the project's installation script

```
$ curl -sL https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh -o install_nvm.sh
```

Make sure the script has been installed

`$ sudo nano install_nvm.sh`

Exit

`ctrl+X`

Run the script

`$ bash install_nvm.sh`

Tell your current session about the changes

`$ source ~/.profile`

Now you can install whichever versions of Node you need

`$ nvm ls-remote`

Install the latest stable build 

`$ nvm install stable`

Double check the installation

`$ node -v`

Now you can use `npm` to install packages. Let's take this opportunity to globally install express

`$ npm i -g express`

# Step 4: Pull in your code
Let's pull in your code from GitHub.

```
$ git clone git@github.com:YourUsername/your-repo.git
```

# Step 5: Install Nginx
First setup a basic firewall on your server

`$ sudo ufw app list`

You should see an output that shows something like:

```
Output
Available applications:
  OpenSSH
```

Make sure the firewall will accept SSH connections
`$ sudo ufw allow OpenSSH`

Enable the firewall
`$ sudo ufw enable`

Type "y" and press ENTER. Check that SSH connections are allowed
`$ sudo ufw status`

```
Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

Install Nginx
```
$ sudo apt-get update
$ sudo apt-get install nginx
```

Next adjust the firewall
`$ sudo ufw app list`

You should get a list of applications once again
```
Output
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```

Enable HTTP since we haven't configured SSL yet
`$ sudo ufw allow 'Nginx HTTP'`

Verify the change
`$ sudo ufw status`

You should see HTTP traffic allowed in the output
```
Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx HTTP                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx HTTP (v6)            ALLOW       Anywhere (v6)
```

Check your web server
`$ systemctl status nginx`
```
Output
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2016-04-18 16:14:00 EDT; 4min 2s ago
 Main PID: 12857 (nginx)
   CGroup: /system.slice/nginx.service
           ├─12857 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
           └─12858 nginx: worker process
```

Looks like the service started successfully, let's test it by sending it real traffic.

You can find your IP easily by typing
```
$ sudo apt-get install curl
$ curl -4 icanhazip.com
```

Once you've ascertained the IP enter it into your browser's address bar
`http://server_domain_or_IP`

You should see the default Nginx page

![nginx default page](/nginx-default.jpg)

# Step 6: Secure Nginx with Let's Encrypt

Install certbot
`$ sudo add-apt-repository ppa:certbot/certbot`

Press ENTER to accept. Then update.
`$ sudo apt-get update`

Install Certbot's Nginx package
`$ sudo apt-get install python-certbot-nginx`

Access your Nginx install
`$ sudo nano /etc/nginx/sites-available/default`

Delete everything in there and replace with what you need
```
server {
    listen 80;
    
    server_name yourdomain.com www.yourdomain.com
}
```

Save, close, and make sure the syntax is correct
`$ sudo nginx -t`

If you get any errors, reopen the file and test again. 

Otherwise, restart the service to load the new configuration
`$ sudo systemctl reload nginx`

Allow HTTPS through the firewall

First check the current settings
`sudo ufw status`

The output will show that only HTTP traffic is currently allowed
```
Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx HTTP                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx HTTP (v6)            ALLOW       Anywhere (v6)
```

Let's also allow HTTPS traffic and delete redundant allowances
```
$ sudo ufw allow 'Nginx Full'
$ sudo ufw delete allow 'Nginx HTTP'
```

Check the status again
`$ sudo ufw status`
```
Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
```

Now we can obtain the SSL certificate with Certbot
`$ sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com`

You may be prompted to enter an email address and agree to the terms of service.

If everything runs successfully you be asked how you want to configure your HTTPS settings.
```
Output
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
-------------------------------------------------------------------------------
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
-------------------------------------------------------------------------------
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

Type "2" and press ENTER.

You should receive a message similare to this:
```
Output
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/example.com/fullchain.pem. Your cert will
   expire on 2017-10-23. To obtain a new or tweaked version of this
   certificate in the future, simply run certbot again with the
   "certonly" option. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

Verify Certbot auto-renewal by running a dry run test
`$ sudo certbot renew --dry-run`

If this runs without errors you are all set.

# Step 7: Setup Nginx as a reverse proxy server
For this part of the tutorial we will assume your node application is using `http://localhost:8080`

Access the configuration file we setup in step 6.
`$ sudo nano /etc/nginx/sites-available/default`

Within the `server` block add the following under your `server_name`:
```
. . .
location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Make sure your syntax is correct
`$ sudo nginx -t`

Restart Nginx
`$ sudo systemctl restart nginx`

# Step 8: Start your app
First we need to install an `npm` package called `pm2`.
`$ npm i -g pm2`

Next go to the local repo where your app's server file is held
`$ cd /path/to/your/repo`

Start the server
`pm2 start index.js`

Now you should be able to type `yourdomain.com` into your browser and see your app.

# Step 9: Re-enable Cloudflare DNS
Back in your Cloudflare account, re-enable the orange cloud. It may take some time to propagate, but you should have a fully secured web app routed through Cloudflare at this point. 

When you go to your domain you will see the green `https` in the address bar.






