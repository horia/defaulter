# defaulter

Default OpenBSD Web Server
```console
server "default" {}
```

![defaulter logo](defaulter.png)

## About
> nice default servers for httpd(8)

The most underused feature of the [`httpd`](https://man.openbsd.org/httpd.8) HTTP daemon is the default server.

The default server is automatically used by httpd(8) in the absence of a custom server configuration.

Using default servers can simplify website hosting.

## Why

* reduce configuration clutter
* simplify webhosting operations
* keep your custom servers or add new ones
* keep your existing apps or add new ones

## How

To add a new website, simply [create a directory](#add-the-document-root-for-wwwexamplecom) and [configure TLS](#configure-tls).

## Features

* default http server to redirect the request scheme "http" to "https"
* opt-in https server to redirect the parent domain to its "www" subdomain
* default https server with dynamic document root

## Getting started

Let's add the `www.example.com` website using default servers.

*please keep reading, a reward for all defaulters' waiting at the end*

Add DNS resource records A[AAA] or CNAME for your domain(s)
```console
example.com.		IN	A	203.0.113.4
example.com.		IN	AAAA	2001:0db8::4
www.example.com.	IN	CNAME	example.com.
```

If needed, include [*pf.conf.defaulter*](src/etc/pf.conf.defaulter) in */etc/pf.conf*
```console
# pf.conf
# Allow traffic on port 80 and 443 to and from the external interface
#
anchor "defaulter" on egress {
  # inbound for relayd
  pass in log proto tcp to (egress) port { http https } \
    keep state (max 500, max-src-conn-rate 100/10)

  # outbound for other services and apps
  pass out log proto tcp from (egress) to port { http https }
}
anchor "relayd/*"
...
```

Install and configure [*httpd.conf*](src/etc/httpd.conf)

Add the parent domain *example.com* alias to redirect it to `www.example.com`:
```console
# httpd.conf
server "defaulter https redirect to www" {
  alias "example.com"
...
```
```sh
rcctl reload httpd
```

#### Add the document root for `www.example.com`
```sh
mkdir /var/www/htdocs/www.example.com
echo Hello > /var/www/htdocs/www.example.com/index.html
```

Install and configure [*relayd.conf*](src/etc/relayd.conf)

To initialize `relayd` without certificates, comment out the relay "https" and "https2" as well as the *tls keypair* statements from [*relayd.conf*](src/etc/relayd.conf)
```sh
rcctl restart relayd
```

#### Configure TLS
```console
# acme-client.conf
domain www.example.com {
	alternative names { example.com }
	domain key "/etc/ssl/private/www.example.com.key"
	domain full chain certificate "/etc/ssl/www.example.com.crt"
	sign with letsencrypt
}
...
```
```sh
acme-client -v www.example.com
ocspcheck -vNo /etc/ssl/example.com.ocsp /etc/ssl/example.com.crt
```

Uncomment the earlier comments from [*relayd.conf*](src/etc/relayd.conf)

Add the `www.example.com` keypair:
```console
# relayd.conf
tls keypair www.example.com
...
```
```sh
rcctl restart relayd
```

Install */etc/daily.local* for daily updates:
```console
# daily.local
name="$(awk '/^[[:space:]]*tls keypair/{printf "%s ",$NF}' /etc/relayd.conf)"
for n in ${name}
 do
  next_part "Let's Encrypt $n"
  acme-client -v $n
  ocspcheck -vNo /etc/ssl/$n.ocsp /etc/ssl/$n.crt
done
rcctl restart relayd
```

For the promised reward, please note that subsequent TLS modifications only involve `acme-client` and changin the *tls keypair* for `relayd`.

