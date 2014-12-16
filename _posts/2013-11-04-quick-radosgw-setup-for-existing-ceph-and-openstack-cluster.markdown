---
layout: post
status: publish
published: true
title: Quick Radosgw setup for existing Ceph and OpenStack cluster.
tags:
- Ceph
- FOSS
- OpenStack
- Howto
comments: []
---
A lot of people are now interested in cloud computing.

A trending solution for providing IaaS platform is OpenStack. With cloud also come the question of storage.

Of course the answer is Ceph, the future of storage.

That's a standard test setup for providing IaaS, installing an Openstack with Ceph for block device.

But wait, as Ceph is installed, why should'nt i benefit from Object Store too?

We will install one machine for this purpose. This machine must already have access to the ceph cluster.

First of all, installing package

We will need a specific version of `libfastcgi` provided by ceph in order to make it work with the horizon dashboard.

We add the repo, then install all the packages :

    sudo echo "deb http://gitbuilder.ceph.com/libapache-mod-fastcgi-deb-precise-x86_64-basic/ref/master precise main" >> /etc/apt/source.list.d/ceph.list
    sudo apt-get update && sudo apt-get install apache2 libapache2-mod-fastcgi

We will then edit the `/etc/ceph/ceph.conf` file to disable the 100-continue, as we want [KISS](http://en.wikipedia.org/wiki/KISS_principle)

We add the folliwing line :

    rgw print continue = false`

Then we enable the necessary modules for apache :

    a2enmod rewrite
    a2enmod fastcgi

We can now install the radosgw :

    sudo apt-get install rados radosgw

Then we add this to `ceph.conf`

    [client.radosgw.gateway]
        host = {host-name}
        keyring = /etc/ceph/keyring.radosgw.gateway
        rgw socket path = /tmp/radosgw.sock
        log file = /var/log/ceph/radosgw.log
        rgw keystone url = {keystone server url:keystone server admin port}
        rgw keystone admin token = {keystone admin token}
        rgw keystone accepted roles = {accepted user roles}
        rgw keystone token cache size = {number of tokens to cache}
        rgw keystone revocation interval = {number of seconds before checking revoked tickets}
        nss db path = /var/lib/ceph/nss

Fill in the various variables according to your existing setup.
Then redeploy your `ceph.conf` to all the node in your cluster.

Create a data directory so radosgw can work :

    sudo mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.gateway

Now we need to create a site configuration for apache :

    <VirtualHost *:80>
    	FastCgiExternalServer /var/www/s3gw.fcgi -socket /tmp/radosgw.sock
            ServerName {fqdn}
            ServerAdmin foo@bar.com
            DocumentRoot /var/www
    	RewriteEngine On
    	RewriteRule ^/([a-zA-Z0-9-_.]*)([/]?.*) /s3gw.fcgi?page=$1&params=$2&%{QUERY_STRING} [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]
            <IfModule mod_fastcgi.c>
                    <Directory /var/www>
                            Options +ExecCGI
                            AllowOverride All
                            SetHandler fastcgi-script
                            Order allow,deny
                            Allow from all
                            AuthBasicAuthoritative Off
                    </Directory>
            </IfModule>
            AllowEncodedSlashes On
            ErrorLog /var/log/apache2/error.log
            CustomLog /var/log/apache2/access.log combined
            ServerSignature Off
    </VirtualHost>

Replace the dummy values by yours (fqdn, email,port).

Now we enable the site :

    a2ensite rgw.conf

If you want to use the 80 port, don't forget to disable the default site :

    a2dissite default

Now we want a fcgi file called `/var/www/s3gw.fcgi` with this content :

    #!/bin/sh
    exec /usr/bin/radosgw -c /etc/ceph/ceph.conf -n client.radosgw.gateway

and make it executable :

    chmod +x /var/www/s3gw.fcgi

We need a keyring so that radosgw can talk to the ceph cluster and add it the cluster auth mechanism

    sudo ceph-authtool --create-keyring /etc/ceph/keyring.radosgw.gateway
    sudo chmod +r /etc/ceph/keyring.radosgw.gateway
    sudo ceph-authtool /etc/ceph/keyring.radosgw.gateway -n client.radosgw.gateway --gen-key
    sudo ceph-authtool -n client.radosgw.gateway --cap osd 'allow rwx' --cap mon 'allow rw' /etc/ceph/keyring.radosgw.gateway
    sudo ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.gateway -i /etc/ceph/keyring.radosgw.gateway

We will create a service and an endpoint into keystone so that he knows about ceph : 

    keystone service-create --name swift --type object-store
    keystone endpoint-create --service-id <id> --publicurl http://radosgw.example.com/swift/v1 --internalurl http://radosgw.example.com/swift/v1 --adminurl http://radosgw.example.com/swift/v1

We also need to create the nss stuff :

    mkdir /var/lib/ceph/nss
    openssl x509 -in /etc/keystone/ssl/certs/ca.pem -pubkey | certutil -d /var/lib/ceph/nss -A -n ca -t "TCu,Cu,Tuw"
    openssl x509 -in /etc/keystone/ssl/certs/signing_cert.pem -pubkey | certutil -A -d /var/lib/ceph/nss -n signing_cert -t "P,P,P"

Restart everything and we are done.

    sudo service ceph restart
    sudo service apache2 restart
    sudo /etc/init.d/radosgw start

Everything should be up and running by now.

Enjoy your self hosted S3/Swift integrated with your Openstack IaaS platform.
