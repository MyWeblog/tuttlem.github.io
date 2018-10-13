---
layout: post
title: Trusting a self-signed certificate
date: 2018-10-12
comments: false
categories: [ "ssl", "trust", "certificate", "linux" ]
---

When working in development and sandboxes, it can make sense to trust the [self-signed certificates](https://en.wikipedia.org/wiki/Self-signed_certificate) that you might be using. This can lower the amount of workflow noise that you might endure.

In today's article, I'll take you through generating a certificate; using the certificate (its use-case is terribly simple), and finally trusting the certificate.

### Generation

In a previous post titled ["Working with OpenSSL"]({% post_url 2015-02-07-working-with-openssl %}), I took you through a few different utilities available to you within the [OpenSSL](https://www.openssl.org/) suite. One of the sections was on generating your own self-signed certificate.

{% highlight bash %}
openssl req -x509 -nodes -days 365 -subj '/C=AU/ST=Queensland/L=Brisbane/CN=localhost' -newkey rsa:4096 -keyout server.key -out server.crt
{% endhighlight %}

You should receive output which looks like the following:

{% highlight plain %}
Generating a RSA private key
.......................................................................................................++++
...............................................................................................................................++++
{% endhighlight %}

On the filesystem now you should have a `server.key` and `server.cer` files waiting for you.

### Using the certificate

Now we're going to stand up a web server that uses this key/certificate pair. Using the [nginx docker image](https://hub.docker.com/_/nginx/), we can quickly get this moving with the following `nginx.conf`.

{% highlight text %}
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  10000;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

  server {
    listen 443;
    index index.html;

    server_name localhost;

    ssl_certificate /opt/server.crt;
    ssl_certificate_key /opt/server.key;

    ssl on;
    root /var/www/public;

    location / {
      try_files $uri $uri/ =404;
    }
  }
}
{% endhighlight %}

Starting the server requires the cerificate, key and configuration file to be mounted in. I've also exposed 443 here.

{% highlight bash %}
docker run --rm \ 
           -ti \
           -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
           -v $(pwd)/server.key:/opt/server.key \
           -v $(pwd)/server.crt:/opt/server.crt \
           -v $(pwd)/index.html:/var/www/public/index.html \
           -p 443:443 \
           nginx
{% endhighlight %}

Right now, when we use the `curl` command without the `--insecure` switch, we receive the following:

{% highlight text %}
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
{% endhighlight %}

### Trusting the certificate

We can now use [cerutil](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSS/tools/NSS_Tools_certutil) to work with the NSS database to add this certificate.

If you're on a brand new system, you may need to create your NSS database. This can be done with the following instructions. Please note, that I'm not using a password to secure the database here.

{% highlight bash %}
mkdir -p %HOME/.pki/nssdb

certutil -N -d sql:$HOME/.pki/nssdb --empty-password
{% endhighlight %}

Some root certificates get added to the store:

{% highlight bash %}
curl -k -o "cacert-root.crt" "http://www.cacert.org/certs/root.crt"
curl -k -o "cacert-class3.crt" "http://www.cacert.org/certs/class3.crt"
certutil -d sql:$HOME/.pki/nssdb -A -t TC -n "CAcert.org" -i cacert-root.crt
certutil -d sql:$HOME/.pki/nssdb -A -t TC -n "CAcert.org Class 3" -i cacert-class3.crt
{% endhighlight %}

The following script can be put into a script file, which will get any certificate from a site and put it into the database for us:

{% highlight bash %}
#!/bin/sh
#
# usage:  import-cert.sh remote.host.name [port]
#
REMHOST=$1
REMPORT=${2:-443}
exec 6>&1
exec > $REMHOST
echo | openssl s_client -connect ${REMHOST}:${REMPORT} 2>&1 |sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p'
certutil -d sql:$HOME/.pki/nssdb -A -t "P,," -n "$REMHOST" -i $REMHOST
exec 1>&6 6>&-
{% endhighlight %}

### Finishing up

Now that you've added your self-signed certificate, you should find that Chrome has no problem navigating to your site. 

The only other issue to now address is [Subject Alternative Name Missing](https://support.google.com/chrome/a/answer/7391219?hl=en) issues in certificates.
