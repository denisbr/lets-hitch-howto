How to secure Varnish with Hitch and Let's Encrypt
==================================================

Introduction
------------

Quote from the https://letsencrypt.org site: "Let’s Encrypt is a new
Certificate Authority: It’s free, automated, and open.". Using Let's
Encrypt anyone with ownership of a domain name can aquire a TLS certificate
for their own personal usage.

There are a number of client-tools available to support this process, and
the project also supplies an official version. However this guide is based
on the very user friendly Acmetool instead, as it simplifies the process and
is available for a number of TLS proxies, including Hitch.

This tutorial will give you instructions for both Ubuntu Xenial (soon to be
released) and CentOS7. At the conclusion, you will have a fully working TLS
setup with automatic certificate renewal.

Prerequisites
-------------

Before starting this tutorial you will need a couple of things.

Firstly you need a working Linux host, either set up with Ubuntu Xenial
(16.04) or CentOS7.
You will need root privileges throughout this tutorial, so either have access
to the root user or sudo privileges (the step-by-step guides assume sudo
usage).

You must own or control a registered domain name that you wish to use the
certificate with. If you do not yet own a domain name, please take a moment to
aquire one from one of the many available registrars. 

When you are in control of a domain name, create an A-record with the name of
the domain that points to the public IP-address of the host you are setting up.
The following guide assumes that this A-record is set up and working, as the
way the certificates are aquired relies on this for validation of domain name
ownership.

In this guide we will use ``example.com`` as the domain name, and we will have
set up both ``example.com`` and ``www.example.com`` to point to our hosts
public IP-address.

Once you have the prerequisites in order, proceed to the actual software setup.

Step 1 - Install Hitch and Varnish
----------------------------------

This step ensures the Hitch and Varnish packages are installed.

Ubuntu Xenial
-------------

Update the package metadata and install the required packages:

```
sudo apt-get update
sudo apt-get install hitch varnish
```

CentOS7 / Red Hat EL7
---------------------

Install the required packages:

```
sudo yum install epel-release
sudo yum install hitch
sudo rpm --nosignature -i https://repo.varnish-cache.org/redhat/varnish-4.1.el7.rpm
```

Step 2 - Configure Varnish
--------------------------

We want Varnish to forward all challenge requests to acmetool, and we are
going to create a request matching rule in VCL that will ensure this forwarding
happens.

The idea is to add this rule in a separate VCL file to not interfer with the
main Varnish VCL.

Again open your favourite editor and create ``/etc/varnish/acmetool.vcl`` with
the following contents:

```
# Forward challenge-requests to acmetool, which will listen to port 402
# when issuing lets encrypt requests

backend acmetool {
    .host = "127.0.0.1";
    .port = "402";
}

sub vcl_recv {
   if (req.url ~ "^/.well-known/acme-challenge/") {
       set req.backend_hint = acmetool;
   }
}
```

Then we need to include this into our main VCL.
Open the file ``/etc/varnish/default.vcl`` and below ``vcl 4.0;`` add:
```
include "/etc/varnish/acmetool.vcl";
```

As we will be using Hitch to forward requests, we want Varnish to listen
to an additional port (6086) using the PROXY protocol support that was added
in Varnish 4.1. 
(If for some reason you do not want to run Varnish 4.1, you can skip this
step, and simply change the port used for Varnish in the hitch config to 6081.)

On Ubuntu Xenial, open the file ``/lib/systemd/system/varnish.service`` 
add ``-a '[::1]:6086,PROXY'`` to the ``ExecStart`` line.

In CentOS7 the same option is added by editing ``/etc/varnish/varnish.params``
and ensure the ``DAEMON_OPTS`` setting includes the following:
``DAEMON_OPTS="-a '[::1]:6086,PROXY'"`` 

Restart Varnish so that it will listen to the new ports, and use the correct
forwarding rule for the challenge requests.
```
sudo service varnish restart
```


Step 2 - Install Acmetool
-------------------------

We will now install the acmetool binaries using the available APT PPA for
Ubuntu, and the unpackaged, pre-built binary for CentOS7.

Ubuntu Xenial
-------------

```
sudo add-apt-repository ppa:hlandau/rhea
sudo apt-get update
sudo apt-get install acmetool
```

CentOS7/Red Hat EL7
-------------------

Download and unpack the latest binary from 
https://github.com/hlandau/acme/releases/latest, and copy it to 
``/usr/local/sbin`` so that it will be found in the PATH.

```
wget https://github.com/hlandau/acme/releases/download/v0.0.41/acmetool-v0.0.41-linux_amd64_cgo.tar.gz
tar xfz acmetool-v0.0.41-linux_amd64_cgo.tar.gz
cp acmetool-v0.0.41-linux_amd64_cgo/bin/acmetool /usr/local/sbin
```

Step 3 - Aquire the certificate
-------------------------------

V, and run the quickstart process. It should detect we are using Hitch and automatically set up a
hook that will generate hitch-compatible cert-packages.


sudo acmetool quickstart
sudo acmetool want lets-hitch.varnish-software.com


References
----------

https://fnord.no/2015/11/12/letsencrypt/
https://github.com/hlandau/acme
https://www.icann.org/registrar-reports/accredited-list.html







Use your favourite editor to create the file ``/etc/hitch/hitch.conf`` and copy the following 
contents into it:

```
## Basic hitch config for use with Varnish and Acmetool

# Listening
frontend = "[*]:443"
ciphers  = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"

# Send traffic to the varnish backend
backend        = "[::1]:6086"
write-proxy-v2 = on

# List of PEM files, each with key, certificates and dhparams
pem-file = ""
```

