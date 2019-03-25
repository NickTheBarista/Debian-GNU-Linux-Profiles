## Using Nginx as SSL tunnel for TCP/UDP service (SSL Mutual Authencitation for normal service)

System: Debian 9

We are using ElasticSearch as our example

### Install ElasticSearch

```
apt install openjdk-8-jdk -y
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-6.x.list
apt update
apt-get install elasticsearch -y
```
changing the binding address as 127.0.0.1

Starting the service
```
systemctl start elasticsearch
```

Using the curl to check if it's working

```
apt install curl libcurl4-gnutls-dev -y
curl http://127.0.0.1:9200
```

And return 

```
{
  "name" : "ktiZiXR",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "H5PnL1q1SSGxcASCfx2c5g",
  "version" : {
    "number" : "6.6.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "1fd8f69",
    "build_date" : "2019-02-13T17:10:04.160291Z",
    "build_snapshot" : false,
    "lucene_version" : "7.6.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

### For server side

Install backports repo

```
sh -c 'printf "deb http://deb.debian.org/debian stretch-backports main" > /etc/apt/sources.list.d/stretch-backports.list'
apt update
apt install -t stretch-backports nginx -y
```

#### Create CA and sign certificate

```
apt install gnutls-bin -y
```

save following command as gen-root-ca.sh
```
#!/bin/bash   

gen_template()
{
cat <<EOF> root-ca.tmpl
cn = "ES Root Certificate Authority"
organization = "x"
unit = "Infrastructure Assurance"
country = "CN"
expiration_days = "720"
ca
signing_key
cert_signing_key
crl_signing_key
EOF
}

gen_ecc_key()
{   
certtool --generate-privkey --ecc --sec-param high -d 2 > root-ca-ecdsa-key.pem   
}

gen_ecc_cert()   
{
certtool --generate-self-signed \
	     --load-privkey root-ca-ecdsa-key.pem \
     --template root-ca.tmpl \
     --outfile root-ca-ecdsa-cert.pem \
     --hash=SHA384
}

gen_template
gen_ecc_key
gen_ecc_cert
```
after running the script.

we got ca certificate `root-ca-ecdsa-cert.pem` and the private key `root-ca-ecdsa-key.pem`

create the server's private key

```
certtool --generate-privkey --rsa --sec-param high > server-rsa-key.pem
```

Create Server's certificate template

```
cat <<EOF> server-cert.tmpl
country = CN
organization = "x"
unit = "ES"
cn = "esserver"
tls_www_client
signing_key
EOF
```

Sign the server's certificate

```
certtool --generate-certificate \
         --load-ca-privkey root-ca-ecdsa-key.pem \
     --load-ca-certificate root-ca-ecdsa-cert.pem \
     --load-privkey server-rsa-key.pem \
     --template server-cert.tmpl \
     --outfile server-rsa-cert.pem \
     --hash=SHA384
```

move the certificate and private key to /etc/ssl/

#### Configurate the Nginx

To be a reverse proxy with ssl on

edit the  `/etc/nginx/nginx.conf` and add the following content at the end of file.

```
stream {
  upstream es_backends {
    server 127.0.0.1:9200;
  }
  server {
    listen 192.168.200.131:9243 ssl;
    ssl_certificate /etc/ssl/server-rsa-cert.pem;
    ssl_certificate_key /etc/ssl/server-rsa-key.pem;
    
    proxy_ssl off;
    proxy_pass es_backends;
  }
}

```
Restart the nginx
```
systemctl restart nginx
```

Check it with curl

```
curl https://192.168.200.131:9343 -k

{
  "name" : "ktiZiXR",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "H5PnL1q1SSGxcASCfx2c5g",
  "version" : {
    "number" : "6.6.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "1fd8f69",
    "build_date" : "2019-02-13T17:10:04.160291Z",
    "build_snapshot" : false,
    "lucene_version" : "7.6.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

So it's working for reverse proxy with nginx

For client authencication we should add our CA certificate and enable the verify `ssl_verify_client on`

now the `/etc/nginx/nginx.conf` should look like below

```
stream {
  upstream es_backends {
    server 127.0.0.1:9200;
  }
  server {
    listen 192.168.200.131:9243 ssl;
    ssl_certificate /etc/ssl/server-rsa-cert.pem;
    ssl_certificate_key /etc/ssl/server-rsa-key.pem;
    
    ssl_client_certificate /etc/ssl/root-ca-ecdsa-cert.pem;
    ssl_verify_client on;

    proxy_ssl off;
    proxy_pass es_backends;
  }
}
```
And using curl to check again

```

curl https://192.168.200.131:9243 -k 
curl: (52) Empty reply from server

```
Because we don't provide the certificate signed by our CA, so we can't access this service

We could using the Server's nginx certificate for testing purpose. Because this certificate signed by our CA.

```
curl --key /etc/ssl/server-rsa-key.pem --cert /etc/ssl/server-rsa-cert.pem https://192.168.200.131:9243 -k

{
  "name" : "ktiZiXR",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "H5PnL1q1SSGxcASCfx2c5g",
  "version" : {
    "number" : "6.6.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "1fd8f69",
    "build_date" : "2019-02-13T17:10:04.160291Z",
    "build_snapshot" : false,
    "lucene_version" : "7.6.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
So the Client authentication is working.


#### Sign the client's certificate

Additional, we have to sign the client's cert

create the client's private key

```
certtool --generate-privkey --rsa --sec-param high > client-rsa-key.pem
```

Create Server's certificate template

```
cat <<EOF> client-cert.tmpl
country = CN
organization = "x"
unit = "ES"
cn = "esclient"
tls_www_client
signing_key
EOF
```

Sign the client's certificate

```
certtool --generate-certificate \
         --load-ca-privkey root-ca-ecdsa-key.pem \
     --load-ca-certificate root-ca-ecdsa-cert.pem \
     --load-privkey client-rsa-key.pem \
     --template client-cert.tmpl \
     --outfile client-rsa-cert.pem \
     --hash=SHA384
```


### For client side

Install backports repo

```
sh -c 'printf "deb http://deb.debian.org/debian stretch-backports main" > /etc/apt/sources.list.d/stretch-backports.list'
apt update
apt install -t stretch-backports nginx -y
```
and copy `root-ca-ecdsa-cert.pem`, `client-rsa-key.pem` and `client-rsa-cert.pem` to client's server /etc/ssl/

Install curl

```
apt install curl libcurl4-gnutls-dev -y
```

Configure the nginx as reserve proxy


edit the  `/etc/nginx/nginx.conf` and add the following content at the end of file.
```
stream {
  upstream es_tls_backends {
    server 192.168.200.131:9243;
  }
  server {
    listen 127.0.0.1:9210;
    proxy_ssl on;
    proxy_ssl_certificate /etc/ssl/client-rsa-cert.pem;
    proxy_ssl_certificate_key /etc/ssl/client-rsa-key.pem;

    proxy_pass es_tls_backends;
  }
}
```

Restart the service
```
systemctl restart nginx
```
Using the Curl to check the Service
```
curl http://127.0.0.1:9210 -k


{
  "name" : "ktiZiXR",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "H5PnL1q1SSGxcASCfx2c5g",
  "version" : {
    "number" : "6.6.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "1fd8f69",
    "build_date" : "2019-02-13T17:10:04.160291Z",
    "build_snapshot" : false,
    "lucene_version" : "7.6.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```





Reference: 

https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/
http://nginx.org/en/docs/stream/ngx_stream_proxy_module.html
https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/
http://nginx.org/en/docs/stream/ngx_stream_ssl_module.html
https://docs.nginx.com/nginx/admin-guide/load-balancer/tcp-udp-load-balancer/
https://www.nginx.com/blog/tcp-load-balancing-udp-load-balancing-nginx-tips-tricks/