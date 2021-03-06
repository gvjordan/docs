---
deprecated: true
author:
  name: Linode
  email: skleinman@linode.com
description: 'Use uWSGI to deploy Python application servers in conjunction with nginx.'
keywords: 'uwsgi,wsgi,nginx,python'
license: '[CC BY-ND 3.0](http://creativecommons.org/licenses/by-nd/3.0/us/)'
alias: ['web-servers/nginx/python-uwsgi/fedora-13/']
modified: Friday, April 29th, 2011
modified_by:
  name: System
published: 'Wednesday, November 10th, 2010'
title: WSGI using uWSGI and nginx on Fedora 13
---



The uWSGI server provides a non-FastCGI method for deploying Python applications with the nginx web server. In coordination with nginx, uWSGI offers great stability, flexibility, and performance. However, to deploy applications with uWSGI and nginx, you must compile nginx manually with the included uwsgi module.

Install uWSGI
-------------

Begin by issuing the following commands to update your system and install dependencies for uWSGI:

    yum update
    yum install python26 python26-devel libxml2 libxml2-devel python-setuptools zlib-devel wget openssl-devel pcre pcre-devel sudo gcc make autoconf automake

Issue the following sequence of commands to download and compile uWSGI:

    cd /opt/
    wget http://projects.unbit.it/downloads/uwsgi-0.9.6.5.tar.gz
    tar -zxvf uwsgi-0.9.6.5.tar.gz
    mv uwsgi-0.9.6.5/ uwsgi/
    cd uwsgi/
    python26 setup.py build
    make

Issue the following command to create a dedicated system user to run the uwsgi processes:

    useradd -M -r --shell /bin/sh --home-dir /opt/uwsgi uwsgi

Send the following sequence of commands to set the required file permissions:

    chown -R uwsgi:uwsgi /opt/uwsgi
    touch /var/log/uwsgi.log
    chown uwsgi /var/log/uwsgi.log

Compile nginx with uWSGI Support
--------------------------------

Issue the following commands to download and compile nginx with support for the `uwsgi` protocol. If you previously installed nginx from packages, remove them at this juncture. The following command sequence mirrors the procedure defined in the [installation guide for nginx](/docs/web-servers/nginx/installation/fedora-13) for compiling nginx from source:

    yum groupinstall "Development Tools"
    yum install zlib-devel wget openssl-devel pcre pcre-devel sudo
    cd /opt/
    wget http://nginx.org/download/nginx-1.0.0.tar.gz
    tar -zxvf nginx-1.0.0.tar.gz
    cd /opt/nginx-1.0.0/
    ./configure --prefix=/opt/nginx --user=nginx --group=nginx --with-http_ssl_module
    make 
    make install
    useradd -M -r --shell /bin/sh --home-dir /opt/nginx nginx
    mkdir /var/lib/nginx
    cp /opt/uwsgi/nginx/uwsgi_params /opt/nginx/conf/uwsgi_params
    wget -O init-rpm.sh http://www.linode.com/docs/assets/654-init-rpm.sh
    mv init-rpm.sh /etc/rc.d/init.d/nginx
    chmod +x /etc/rc.d/init.d/nginx
    chkconfig --add nginx
    chkconfig nginx on
    /etc/init.d/nginx start 

Configure uWSGI
---------------

Issue the following command to download an init script to manage the uWSGI process, located at `/etc/init.d/uwsgi`:

    cd /opt/
    wget -O init-rpm.sh http://www.linode.com/docs/assets/653-uwsgi-init-rpm.sh
    mv /opt/init-rpm.sh /etc/init.d/uwsgi
    chmod +x /etc/init.d/uwsgi

Create an `/etc/default/uwsgi` file to specify specific settings for your Python application. The `MODULE` specifies the name of the Python module that contains your `wsgi` specification. Consider the following example:

{: .file-excerpt }
/etc/default/uwsgi
:   ~~~ bash
    PYTHONPATH=/srv/www/ducklington.org/application
    MODULE=wsgi_configuration_module
    ~~~

If you want to deploy a "Hello World" application, insert the following code into the `/srv/www/ducklington.org/application/wsgi_configuration_module.py` file:

{: .file }
/srv/www/ducklington.org/application/wsgi\_configuration\_module.py
:   ~~~ python
    import os
    import sys

    sys.path.append('/srv/www/ducklington.org/application')

    os.environ['PYTHON_EGG_CACHE'] = '/srv/www/ducklington.org/.python-egg'

    def application(environ, start_response):
        status = '200 OK'
        output = 'Hello World!'

        response_headers = [('Content-type', 'text/plain'),
                            ('Content-Length', str(len(output)))]
        start_response(status, response_headers)

        return [output]
    ~~~

Issue the following commands to make this init script executable, ensure that uWSGI is restarted following the next reboot sequence, and start the service:

    chkconfig --add uwsgi
    chkconfig uwsgi on
    /etc/init.d/uwsgi start 

Configure nginx Server
----------------------

Create an nginx server configuration that resembles the following for the site where the uWSGI app will be accessible:

{: .file-excerpt }
nginx virtual host configuration
:   ~~~ nginx
    server {
        listen   80;
        server_name www.ducklington.org ducklington.org;
        access_log /srv/www/ducklington.org/logs/access.log;
        error_log /srv/www/ducklington.org/logs/error.log;

        location / {
            include        uwsgi_params;
            uwsgi_pass     127.0.0.1:9001;
        }

        location /static {
            root   /srv/www/ducklington.org/public_html/static/;
            index  index.html index.htm;
        }
    }
    ~~~

All requests to URLs ending in `/static` will be served directly from the `/srv/www/ducklington.org/public_html/static` directory. Restart the web server by issuing the following command:

    /etc/init.d/nginx restart

Additional Application Servers
------------------------------

If the Python application you've deployed requires more application resources than a single Linode instance can provide, all of the methods for deploying a uWSGI application server are easily scaled to rely on multiple uSWGI instances that run on additional Linodes with the request load balanced using nginx's `upstream` capability. See our documentation of [proxy and software load balancing with nginx](/docs/websites/nginx/basic-nginx-configuration/front-end-proxy-and-software-load-balancing) for more information. For a basic example configuration, see the following example:

{: .file-excerpt }
nginx configuration
:   ~~~ nginx
    upstream uwsgicluster {
         server 127.0.0.1:9001;
         server 192.168.100.101:9001;
         server 192.168.100.102:9001;
         server 192.168.100.103:9001;
         server 192.168.100.104:9001;
    }

    server {
        listen   80;
        server_name www.ducklington.org ducklington.org;
        access_log /srv/www/ducklington.org/logs/access.log;
        error_log /srv/www/ducklington.org/logs/error.log;

        location / {
            include        uwsgi_params;
            uwsgi_pass     uwsgicluster;
        }

        location /static {
            root   /srv/www/ducklington.org/public_html/static/;
            index  index.html index.htm;
        }
    }
    ~~~

In this example, we create the `uwsgicluster` upstream, which has five components. One runs on the local interface, and four run on the local network interface of distinct Linodes (the `192.168.` addresses or the private "back end" network). The application servers that run on those dedicated application servers are identical to the application servers described above. However, the application server process must be configured to bind to the appropriate network interface to be capable of responding to requests.

More Information
----------------

You may wish to consult the following resources for additional information on this topic. While these are provided in the hope that they will be useful, please note that we cannot vouch for the accuracy or timeliness of externally hosted materials.

- [Installing Nginx on Fedora 13](/docs/web-servers/nginx/installation/fedora-13)
- [Deploy a LEMP Server on Fedora 13](/docs/lemp-guides/fedora-13/)
- [Configure nginx Proxy Servers](/docs/websites/nginx/basic-nginx-configuration/front-end-proxy-and-software-load-balancing)



