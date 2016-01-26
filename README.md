# How to secure Varnish with Hitch and Let's Encrypt

## Introduction

Quote from the https://letsencrypt.org site: "Let’s Encrypt is a new
Certificate Authority: It’s free, automated, and open.". Using Let's
Encrypt anyone with ownership of a domain name can aquire a TLS certificate
for their own personal usage.

There are a number of client-tools available to support this process, and
the project also supplies an official version. However this guide is based
on the very user friendly [Acmetool](https://hlandau.github.io/acme/) instead,
as it simplifies the process and is available for a number of TLS proxies,
including [Hitch](https://hitch-tls.org/).

This tutorial will give you instructions for both Ubuntu 16.04 Xenial (soon
to be released) and CentOS7. At the conclusion, you will have a fully working
TLS setup with automatic certificate renewal.

## Prerequisites

Before starting this tutorial you will need a couple of things.

Firstly you need a working Linux host, either set up with Ubuntu Xenial
or CentOS7.

You will need root privileges throughout this tutorial, so either have access
to the root user or sudo privileges (the step-by-step guides assume sudo
usage).

You must own or control a registered domain name that you wish to use the
certificate with. If you do not yet own a domain name, please take a moment to
aquire one from one of the many available registrars. 
(See [Icann.org](https://www.icann.org/registrar-reports/accredited-list.html)
for an exhaustive list.)

When you are in control of a domain name, create an A-record with the name of
the domain that points to the public IP-address of the host you are setting up.
The following guide assumes that this A-record is set up and working, as the
way the certificates are aquired relies on this for validation of domain name
ownership.

In this guide we will use ``example.com`` as the domain name, and we will have
set up both ``example.com`` and ``www.example.com`` to point to our hosts
public IP-address.

Once you have the prerequisites in order, proceed to the actual software setup.

## Step 1 - Install Hitch and Varnish

This step ensures the Hitch and Varnish packages are installed.

### Ubuntu Xenial

Update the package metadata and install the required packages:

```
sudo apt-get update
sudo apt-get install hitch varnish
```

### CentOS7 / Red Hat EL7

Install the required packages:

```
sudo yum install epel-release
sudo yum install hitch
sudo rpm --nosignature -i https://repo.varnish-cache.org/redhat/varnish-4.1.el7.rpm
```

## Step 2 - Configure Varnish

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
Open the file ``/etc/varnish/default.vcl`` and add this below your backend
definitions:
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


## Step 3 - Install Acmetool

We will now install the acmetool binaries using the available APT PPA for
Ubuntu, and the unpackaged, pre-built binary for CentOS7.

### Ubuntu Xenial

```
sudo add-apt-repository ppa:hlandau/rhea
sudo apt-get update
sudo apt-get install acmetool
```

### CentOS7/Red Hat EL7

Download and unpack the latest binary from 
https://github.com/hlandau/acme/releases/latest, and copy it to 
``/usr/local/sbin`` so that it will be found in the PATH.

```
wget https://github.com/hlandau/acme/releases/download/v0.0.41/acmetool-v0.0.41-linux_amd64_cgo.tar.gz
tar xfz acmetool-v0.0.41-linux_amd64_cgo.tar.gz
sudo cp acmetool-v0.0.41-linux_amd64_cgo/bin/acmetool /usr/local/sbin
```

## Step 4 - Aquire the certificate

Now we will use acmetool to aquire a certificate. 

Now we have everything in place and we run the acmetool quickstart process.
It should detect we are using Hitch and automatically set up a hook that
will generate hitch-compatible certificate-packages from certificate
requests.

```
sudo acmetool quickstart
```

Answer the prompts like this to enable live certificates authenticated through
challenge requests proxied through Varnish.

```
------------------------- Select ACME Server -----------------------
1) Let's Encrypt (Live) - I want live certificates
```

```
----------------- Select Challenge Conveyance Method ---------------
2) PROXY - I'll proxy challenge requests to an HTTP server
```

```
----------------- Install HAProxy/Hitch hooks? ---------------------
Yes) Do you want to install the HAProxy/Hitch notification hook?
```

Before we continue to requesting our certificate we need to generate a Diffie-
Hellman group file (aka dhparams), used for perfect forward secrecy. 

```
sudo openssl dhparam -out /var/lib/acme/conf/dhparams 2048
```

Now we can finally get our certificate:

```
sudo acmetool want example.com
```

## Step 5 - Configure Hitch

Now we should have our own valid certificate, and we can use it to set up
Hitch. As previously mentioned we configured Varnish to listen to an additional
port (6086) where it will accept requests using the PROXY protocol.

Use your favourite editor to create the file ``/etc/hitch/hitch.conf`` and
copy the following contents into it:

```
## Basic hitch config for use with Varnish and Acmetool

# Listening
frontend = "[*]:443"
ciphers  = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"

# Send traffic to the Varnish backend using the PROXY protocol
backend        = "[::1]:6086"
write-proxy-v2 = on

# If you run Varnish 4.0 use this instead
#backend        = "[::1]:6081"
#write-proxy-v2 = off 

# List of PEM files, each with key, certificates and dhparams
pem-file = "/var/lib/acme/live/example.com/haproxy"
```

Start hitch with the new configuration:

```
sudo service hitch start
```

