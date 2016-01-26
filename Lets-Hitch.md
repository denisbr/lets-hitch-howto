Ubuntu Xenial:

```
sudo apt-get install hitch varnish
sudo add-apt-repository ppa:hlandau/rhea
sudo apt-get update
sudo apt-get install acmetool
sudo acmetool quickstart
sudo acmetool want lets-hitch.varnish-software.com
```

* Edit ``/lib/systemd/system/varnish.service`` add ``-a '[::1]:6086,PROXY'`` to ExecStart
* Create ``/etc/hitch/hitch.conf`` with contents:

```
## Basic hitch config for use with Varnish and Acmetool
## https://github.com/hlandau/acme

# Listening
frontend = "[*]:443"
ciphers  = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"

# Send traffic to the varnish backend
backend        = "[::1]:6086"
write-proxy-v2 = on

# List of PEM files, each with key, certificates and dhparams
pem-file = "/etc/hitch/lets-hitch.varnish-software.com-combined.pem"
```

* Create ``/etc/varnish/acmetool.vcl`` with contents:
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

* Edit ``/etc/varnish/default.vcl`` add ``include /etc/varnish/acmetool.vcl``
