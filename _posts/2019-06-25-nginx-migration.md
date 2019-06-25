---
layout: post
title: "Adding an Nginx Reverse Proxy to a Live Server"
date: 2019-06-25 14:08:16 +0200
tags: devops
---

I host a few small projects using PHP and Apache which I run on a single cheap virtual server. That setup served me well over the years, until I wrote something in Node.js.

In the fear (or hope?) that one of my projects might suddenly become popular, I tend to use different (sub-) domains for individual services, so I could easily move them to a different server by just changing a DNS record. I wanted to keep this system, while also keep using the standard ports for http and https. The obvious solution was to route all web traffic through a reverse proxy. 

There are plenty of [excellent](https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-as-a-reverse-proxy-for-apache) [guides](https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-as-a-web-server-and-reverse-proxy-for-apache-on-one-ubuntu-18-04-server) on how to setup an Nginx / Apache combination. However, my server is hosting multiple services that people are actively using and could even lead to data loss when offline for longer than a few minutes. They also use multiple ssl certificates, which further complicates the situation.

The solution I came up with manages to add the reverse proxy with practically no downtime. Figuring everything out involved a lot of trail and error, and I hope this guide will help you reach the same goal more directly.

I created a [GitHub repo](https://github.com/stepmuel/nginxmig) with some of the scripts I used for the migration.

# Outline

* Configure Apache to listen to `port 85` in addition to the regular ports.
* Install Nginx as reverse proxy from `port 81` to `http://localhost:85`.
* Add Nginx configuration for custom sites
  * Add subdomains, existing certificates, and use `port 444` for ssl.
* Test if all sites work as intended with their *testing port* (actual port + 1).
  * Are the sites reachable using ssl? Try `https://domain:444`.
  * Are all subdomains working? Try `http://www.domain:81`.
* Perform the *port switch*. (***critical moment***)
  * Configure Apache to only listen to `port 85` (also disable ssl port).
  * Configure Nginx to replace their test ports with the actual ports.
  * Make Apache use the new configuration, immediately followed by Nginx.
* Test if everything works as intended.
  * If not, `panic.sh` will terminate Nginx and make Apache use the original ports.

# Before You Start

I tried all my commands in a local virtual machine, and waited until I had a solid plan before running anything on my live server. I even had to cancel the migration a few times and adjust my testing setup because my server didn't behave as expected. I used [Vagrant](https://www.vagrantup.com/) and the image `ubuntu/xenial64` (since my server is running Ubuntu 16.04). The VirtualBox snapshot feature was very useful to revert back to a known state and try a different approach while avoiding unwanted side effects. The provided `Vagrantfile` will install Apache which can be reached at `http://55.55.55.5/`.

I didn't want to spend more than a day on the migration, and could deal with a few minutes of downtime, so I only tried to simulate parts that seemed risky and I was unsure about. How you want to balance risk vs. time is ultimately up to you. You won't be able to perfectly replicate the live environment, but there is some value in not having to do it for the first time on a live server.

In addition to my virtual test bed, I took some more precautions:

* Use shell scripts to reduce simple mistakes like typos or missing commands.
* Regularly check if everything is in the state you expect it to be.
  * While I did this manually, this could also be automated (maybe even with automatic rollback)
* Design your process / steps to be stable.
  * E.g. running a step twice or without sudo shouldn't have bad consequences.
  * Test safe but realistically (test ports). Try to make "going live" a simple action (port switch).
* Always be prepared to abort and roll everything back. (Have a plan!)
  * Since things might get hectic, a script might be helpful.

Adding a reverse proxy also has a few drawbacks you might not have thought of. The *Proxy Configuration* section lists some of them. 

# Preparing Apache

In the final setup, all requests will be forwarded to Apache via the port 85. Making Apache *also* listen to that port won't impact the existing sites, and we can use it to test our Nginx setup along the way. That's why this is our first step.

First, open a new browser window, and load every single site you are hosting in a new tab. Use all subdomains (with and without www, if available). Also use both http and https, if available.

If everything loads, you are now in a known state. If something isn't working, you know that it wasn't caused by the new configuration we are about to do. Keep the window open to test if everything still works after the configuration. You might want to use a feature like "Bookmark All Tabs" which might allow you to reload all sites at once.

I assume all your sites are represented by files in `/etc/apache2/sites-available` that define a `VirtualHost`. Add port 85 to all hosts you want to keep using behind Nginx. Only modify the non-ssl variant (port 80), even if you want to access the site over https (ssl termination will later be handled by Nginx).

```
# Change this:
<VirtualHost *:80>
# To this:
<VirtualHost *:80 *:85>
```

The `port-switch-apache.sh` script can automate this process.

```sh
sudo ./port-switch-apache.sh domain1.conf [domain2.conf] ...
```

Then add `Listen 85` to `/etc/apache2/ports.conf`, so the Apache handles incoming http requests to port 85. Make a backup of the original config file first. (The backup will be used by the `panic.sh` script.)

```sh
sudo cp /etc/apache2/ports.conf /etc/apache2/ports.conf.default
sudo echo "Listen 85" | sudo tee -a /etc/apache2/ports.conf
```

Make sure that all modified configuration files are correct. `ports.conf` must still contain the original ports (probably 80 and 443). The site configurations must still be valid and listen to both ports, old and new. If something goes wrong, the configuration files have to be restored / fixed manually.

If you think everything is fine, run `sudo service apache2 graceful`. Then check if the old sites still work by reloading all tabs on the window we prepared at the beginning of this section.

If everything works, also try to access all domains with the newly added port 85.

# Installing Nginx

If all our sites are accessible using port 85, we can now install Nginx. This is a bit more complicated than it seems, because dpkg will try to set it up as the default web server, listening to port 80. Since the port is already used by Apache, the package configuration will fail at the part where it tries to start the server. 

This part is a bit hacky, but I couldn't find a better solution online. I let the installation fail, remove the default site (the one asking for port 80), then restart the package configuration to finish the installation. I recommend to test for yourself if installing Nginx affects Apache on the Linux distribution you are using. This was the main reason I used a local VM.

The script `nginx-install.sh` handles that process. It also installs and enables a new default site which redirects requests to port 81 to port 85, and a configuration snippet for some common proxy arguments.

Once Nginx is up and running, all existing Apache sites should be accessible using the port 81. Please verify that all tabs from our test window are still working.

The Nginx installation script adds the following configuration files during the setup process:

Content of `/etc/nginx/sites-available/apache-default` (also linked to `sites-enabled`):

```
server {
    listen 81 default_server;
    server_name _;
    location / {
        include snippets/proxy-nobuff.conf;
        proxy_read_timeout 1h;
        proxy_pass http://localhost:85;
    }
}
```

Content of `/etc/nginx/snippets/proxy-nobuff.conf`:

```
# Add useful proxy headers
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# Disable buffering
proxy_buffering off;
proxy_request_buffering off;
```

# Proxy Configuration

Putting a reverse proxy in front of Apache doesn't come without consequences:

Some `$_SERVER` variables inside PHP are no longer valid, notably `REMOTE_ADDR`, `SERVER_PORT`, and `HTTPS`. The can be replaced by some of the new headers we set (see example).

```php
// Local nginx proxy
$remoteIP = $_SERVER['REMOTE_ADDR'];
if ($_SERVER['REMOTE_ADDR'] === '127.0.0.1' && isset($_SERVER['HTTP_X_REAL_IP'])) {
    $remoteIP = $_SERVER['HTTP_X_REAL_IP'];
}
```

Nginx has a default *read* timeout of 1 minute. This means, it will cancel a connection if the PHP script doesn't write any new data for more than that period of time. Since Apache and PHP have their own timeouts, this might not be wanted. I for example manually extend the timeout in certain scripts to perform long maintenance tasks. Nginx can't be configured to not have a timeout, so I set it to 1 hour, which is enough for me.

Nginx uses buffering in order to minimize the time a process is busy handling a request. Since the client might have a slow internet connection, transferring the request data might take a long time. By default, Nginx will store the incoming request and initiate the proxy request only after all data is received. The same thing goes for the server response: Nginx stores all data locally, and won't start sending anything back to the client before the proxy has sent all data.

In some cases, this behavior adds latency and will significantly alter server communication. For example, one of my scripts returns large files, line by line. In this case, Nginx adds a very long delay before the client receives the first line. In the config files above, I disabled both input- and output buffering for proxy requests, with the goal of making the system behave more like a regular Apache server.

# Add Nginx Sites and Certificates

If you are just using unencrypted http and are happy with having Apache as the default server, you can skip this step. If some of your sites are accessed using https, you'll have to add the certificates to the Nginx configuration. Nginx will then handle ssl termination, and delegate the request over port 85, unencrypted. In my case I use [Let's Encrypt](https://letsencrypt.org/) certificates and set everything up using [Certbot](https://certbot.eff.org/). My configuration relies heavily on this setup, and might have to be adjusted if your looks differently.

If you use Certbot to configure Nginx, it will add a shared configuration at `/etc/letsencrypt/options-ssl-nginx.conf` to all sites. Unfortunately, this file doesn't exist if the Nginx module is missing, which will be the case if you follow the instructions on the Certbot website. To install it retroactively, use

```sh
# Install Nginx module for Certbot
sudo apt-get install certbot python-certbot-nginx
# Create options-ssl-nginx.conf file
sudo certbot --nginx
# Cancel with `c` on the first prompt
```

Once the config file is available, my site template `apache-le-ssl` can be used:

```
server {
    server_name {$name};
    location / {
        include snippets/proxy-nobuff.conf;
        proxy_read_timeout 1h;
        proxy_pass http://localhost:85;
    }

    listen 81; # managed by Certbot

    listen 444 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/{$name}/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/{$name}/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
```

You can use the provided script `sudo ./add-site.sh domain.name.net` to create new Nginx sites. It will

* replace the `{$name}` placeholder with the given domain string,
* copy the resulting file into `/etc/nginx/sites-available`,
* and link it to `sites-enabled`.

It links to certificate stored at `/etc/letsencrypt/live/{$name}`, which is where Let's Encrypt certificates tend to be stored. All Nginx sites are configured to listen to port 81 or 444 (for now), so they can be tested without interfering with the live sites served by Apache.

Once all your sites are added, run `sudo service nginx reload` to enable them. You should now be able to access them using `https://domain.name.net:444`. Verify that all sites are loading and the certificates are valid.

# The Port Switch

At this point, our work is almost done. All that's left to do is remove the ports 80 and 443 from Apache and give them to Nginx. But before you start, a few last safety checks:

* Check one last time that all tabs from your test window will load when using the ports 81 for http and 444 for https.
* If you want, you can now test the `panic.sh` script.
  * After running to script, all sites should be accessible using their original ports, but not the +1 test ports.
  * Read the next section how to restore the previous state after running `panic.sh`.

The script `sudo ./port-switch-nginx.sh siteconfig [siteconfig] ...` can be used to change ports 81 to 80, and 444 to 443 in Nginx configuration files. You should be able to simply run `sudo ./port-switch-nginx.sh /etc/nginx/sites-available/*`. Verify that all config files are indeed using only the correct port numbers (80 and 443) before proceeding.

Now it's time to switch everything over to Nginx. The `switch.sh` script will change `ports.conf` to *only* listen to the port 85. Then it will restart Apache (at which point nothing is listening on port 80 or 443 anymore; all new requests will be stalled). Nginx will load the new configuration and start handling those request immediately after. And we're done!

One last time, check if all sites are running as intended.

# Panic!

`panic.sh` will try to reset all configuration to a known working state in case something goes wrong during the *switch* process. It will disable Nginx and restore the original `ports.conf` which we saved in the *Preparing Apache* section.

I designed `panic.sh` to remove as much friction and potential user error as possible. for example, it will ask for the password in case the user forgets to use `sudo`.

After you invoked `panic.sh`, perform the following steps to get ready for another switch attempt:

* Add `Listen 85` to be re-added to `ports.conf` again.
* Make sure all configurations in `/etc/nginx/sites-available` are using ports 81 and 444 instead of 80 and 443.
* Restart Nginx with `service nginx start`.

# Fix Certbot Auto Renew

If you have Certbot configured to automatically renew the certificates, it will no longer be able to do so, since Apache is no longer handling any ssl requests. This can be fixed pretty easily by manually changing the config files at `/etc/letsencrypt/renewal/*.conf`.

```sh
# Change this line:
authenticator = apache
# to this line:
authenticator = nginx
```

Run `sudo certbot renew --dry-run` to make sure everything works.

Thanks for reading!
