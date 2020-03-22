---
title: Webhooks Continued, Server Configuration And You
taxonomy:
	tag: [Git, Websites, Webhooks, Linux, Nginx, LetsEncrypt]
media_order: red_and_white_plastic_toy-scopio-bea1e50d-ed7a-4295-9ca8-ed96298b80bf.jpg
---

Continued from [Using Git to Keep Your Website up to Date](/blog/websitewebhookupdate)

In the previous post, we set up a listener that would listen for an HTTP request from Github and run a script to update your website automatically. But, there are a few things left undone. 

In this phase, we'll set up an [Nginx](https://www.nginx.com/) reverse proxy and use the amazing service [LetsEncrypt](https://letsencrypt.org/) that will let us do the following:

1. Throttle the number of requests allowed
2. Enable HTTPS for the script

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

## Throttling Traffic
It's unlikely that your website needs to be updated many times a second. So to help maintain security, let's make sure that requests are only passed to the webhook update script at a throttled speed, so that the script won't activated too often. 

