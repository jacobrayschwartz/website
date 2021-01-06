---
title: Webhooks Continued, Server Configuration And You
taxonomy:
	tag: [Git, Websites, Webhooks, Linux, Nginx, LetsEncrypt]
media_order: red_and_white_plastic_toy-scopio-bea1e50d-ed7a-4295-9ca8-ed96298b80bf.jpg
date: 3/30/2020
---

Continued from [Using Git to Keep Your Website up to Date](/blog/websitewebhookupdate)

In the previous post, we set up a listener that would listen for an HTTP request from Github and run a script to update your website automatically. But, there are a few things left undone. 

In this phase, we'll set up an [Nginx](https://www.nginx.com/) reverse proxy and use the amazing service [LetsEncrypt](https://letsencrypt.org/) that will let us do the following:

1. Enable HTTPS for the script
2. Throttle the number of requests allowed

## Why
Three words: security, security, security. If a website is public, you should assume some malicious actor will try to take advantage of it. You want to ensure as much of your website uptime as you can, and ensure the safety of your visitors. 

By throttling the number of requests allowed, it makes it that much harder for attackers to get leverage on your site. Throttling requests follows the same concept that password hashes use. Eventually, every hash can be broken, but if the hash takes a long time computationally, it takes much longer for it to be broken. 

You should always look to encrypt traffic for your visitors. This helps keep your data and their data safe. The same is true for all requests coming into your site. Encrypted traffic means that some attacker can't see the data coming in to find some vulnerability. 

## Setting up Nginx
First, we need to install Nginx. You probably already have Nginx installed if you're running Grav. If you're using Apache instead of Nginx, [Take a look at this article from Ben Dellar](https://bendellar.com/apache-as-reverse-proxy-for-letsencrypt-free-https-certificates/). If you haven't installed either for some reason, you can do it now! So grab your favorite package manager and run the install. For Ubuntu, it should look something like this:

```bash
sudo apt install nginx
```

Before we write the configuration for the reverse proxy, let's get some information. We'll need the following to set up the configuration:

1. Port the webhook app is running on - **AppPort**
2. Domain name of your website (no tld required) - **Domain**
3. External port that Github will send the webhook to - **ExternalPort**

Now, we need to configure the reverse proxy service. Change directory into the Nginx install. This will likely be `/etc/nginx`, but might be a bit different depending on your installation. 

Change directory again to `sites-available`. This directory contains all available configurations for Nginx, but they are not enabled until a symlink for them exists in `sites-enabled`. 

Create a file called `webhook-reverse-proxy.conf` with the following content. Be sure to replace any of the items in between the angle braces `<` and `>`:

*webhook-reverse-proxy.conf*
```
server {
        set $upstream 127.0.0.1:<AppPort>;
        server_name <Domain>:<ExternalPort>;
        location / {
                proxy_pass_header Authorization;
                proxy_pass http://$upstream;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_http_version 1.1;
                proxy_set_header Connection "";
                proxy_buffering off;
                client_max_body_size 0;
                proxy_read_timeout 36000s;
                proxy_redirect off;
        }

        listen <ExternalPort>;
}
```

This configuration takes the traffic from `<ExternalPort>` and passes it with all of the appropriate header information to the service running on `<AppPort>`. For instance, your `<AppPort>` might be `9090` but your external port would be something else like `5555`. You should take a look at your currently used ports by running `sudo netstat -tulpn | grep LISTEN`. Head on over to check the list of registered ports over at [IANA](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) to make sure you don't try to take ports used by some other service. 

Once you've set up your configuration, make a symbolic link to it inside of the `sites-available` directory.

```bash
sudo ln -s /etc/nginx/sites-enabled/webhook-reverse-proxy.conf /etc/nginx/sites-available/webhook-reverse-proxy.conf
```

Finally, tell Nginx to reload the configuration.

```bash
sudo nginx -s reload
```

Make sure the webhook settings for your repository on Github match the external port defined in the Nginx config. 

You should now be able to push to your repository and see updates reflected in your website. 




## Enabling HTTPS
Securing your traffic is always a good thing. If your connection to your clients is encrypted, it's one less way that an attacker can do something you don't want them to do. Now, with Let's Encrypt, *it's so easy that it'd be criminal not to.*

For some great information on how HTTPS actually works, [I recommend checking out this video on Youtube.](https://www.youtube.com/watch?v=T4Df5_cojAs), we just won't be creating our own CA, because no browser will trust it. Instead, we'll request that some trusted CA sign our certificate. 

Before Let's Encrypt, the state of HTTPS was fractured. You had to pay an Certificate Authority like Verisign to be able to get a certificate file, then read through arcane documentation to be able to enable HTTPS. 

Now, Let's Encrypt allows you to create the certificate for free! With [certbot](https://certbot.eff.org/), you can easily create a certificate, get it signed by the Let's Encrypt CA and then apply the certificate to just about any setup. We'll go over how to apply the certificate to the reverse proxy we've set up before, but you should also add a certificate to your actual website. It's just about the same process, especially if your website is running on Nginx as well. 

Note, the documentation over at [Let's Encrypt](https://letsencrypt.org/getting-started/) is wonderful. I'll go over only the basics of how to add the certificate in our setup. 

First, install certbot. Head on over to (https://certbot.eff.org/) and click on the Get Certbot instructions button. Follow along with the prompts, make sure to select **Nginx** for software and the correct OS your server is running. For example, I chose Nginx running on Ubuntu 18.04 LTS.


![Select 'Nginx' and your operating system](certbot_selection.gif)

Follow the provided instructions to install certbot.

Now, run certbot passing the `--nginx` flag:

```bash
$ sudo certbot certonly --nginx
```

This will create a certificate for your site. Your domain should automatically be selected. At the end, you should see some output like so:

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/<domain>/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/<domain>/privkey.pem
   Your cert will expire on 2020-06-20. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

Take note of that path `/etc/letsencrypt/live/<domain>/` where `<domain>` should be your domain. We'll use that to tell Nginx where to look for the certificate files.

Now, open your `webhook-reverse-proxy.conf` file. We need to make the following changes:

1. Tell Nginx to use ssl on the listening port
2. Give nginx the information on the certificates

For step 1, edit the line that says `listen <ExternalPort>;`, change it to `listen <ExternalPort> ssl;`.

For step 2, add the following lines after the listen directive, before the closing brace for the `server` block, be sure to replace the paths to the script with the ones given to you from the script:

*webhook-reverse-proxy.conf*
```
listen <ExternalPort> ssl; #Be sure this has the ssl parameter
ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem; # managed by Certbot
ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem; # managed by Certbot
include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
``` 

Note, you should have already had the `listen <ExternalPort>` directive, but now, you need to add the `ssl` parameter to it. 


## Throttling Traffic
It's unlikely that your website needs to be updated many times a second. So to help maintain security, let's make sure that requests are only passed to the webhook update script at a throttled speed, so that the script won't activated too often. 

First, we need to create the limit definition. So, add this directive to the top of your configuration file, above the server block:

```
limit_req_zone $binary_remote_addr zone=one:32k rate=1r/s;
```

> **Command Definition**
>
> The command above defines a new shared memory zone that tracks the requests from different clients. 
>
> * `$binary_remote_addr` tells Nginx to store the addresses of clients in binary. This is used instead of something like `$remote_addr` (a string representation) to save on space.
>
> * `zone=one:1m` tells Nginx to create a memory zone called `one` and give it a size of 32 Kilobytes. This shared space doesn't need to be large because only one client should be accessing this endpoint. 
>
> * `rate=1r/s` tells Nginx to only allow one request per second, per client. 

Next, add the configuration to the location block, telling Nginx to limit requests on that endpoint using the definition above:

*webhook-reverse-proxy.conf*
```
limit_req_zone $binary_remote_addr zone=one:32k rate=1r/s;

server {
        set $upstream 127.0.0.1:<AppPort>;
        server_name <Domain>:<ExternalPort>;
        location / {
                limit_req zone=one;
                proxy_pass_header Authorization;
                ...
        }
        listen <ExternalPort> ssl;
        ...
}
```

That's it! Now, reload your configuration using the Nginx console command:

```bash
sudo nginx -s reload
```

Finnaly, your config file should look like the following:

*webhook-reverse-proxy.conf*
```
limit_req_zone $binary_remote_addr zone=one:32k rate=1r/s;

server {
        set $upstream 127.0.0.1:<AppPort>;
        server_name <Domain>:<ExternalPort>;
        location / {
                limit_req zone=one;
                proxy_pass_header Authorization;
                proxy_pass http://$upstream;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_http_version 1.1;
                proxy_set_header Connection "";
                proxy_buffering off;
                client_max_body_size 0;
                proxy_read_timeout 36000s;
                proxy_redirect off;
        }

        listen <ExternalPort> ssl; # managed by Certbot
        ssl_certificate /etc/letsencrypt/live/<Domain>/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/<Domain>/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
```

## Conclusion
That's it for this server configuration. In this article, we set up an Nginx reverse proxy, used Let's Encrypt and Certbot to enable HTTPS on the webhook port, and throttled the requests to activate our webhook.

All applications should be designed with security in mind. Never assume your user base is too small or that your data is not useful to someone else. With the advent of simple tools like Certbot, there's no excuse for not securing your website behind HTTPS. It's your obligation to do it for your users. 

I hope this information is useful!