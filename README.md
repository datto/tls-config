Datto's Secret TLS Sauce
========================

As with most cloud service providers, security is Kind Of A Big Deal at Datto. With all the FUD going around about the NSA breaking SSL/TLS and stuff, it was a big priority for me to ensure that communications between Datto employees and devices and our mothership are secured.

The lowest hanging fruit for SSL/TLS experimentation was our mostly-internal LDAP/Kerberos account management interface, since only employees and a few distributors use it, and therefore we aren't as afraid of breaking older browsers.

If you just want to see results, [here's the test on sso.datto.net](https://www.ssllabs.com/ssltest/analyze.html?d=sso.datto.net&hideResults=on).

Why?
====

There is [a lot of FUD](http://www.zdnet.com/article/has-the-nsa-broken-ssl-tls-aes/) going around on the Internet that the NSA and similar organizations have a vast reserve of zero-day attacks on TLS. Even if they can't decrypt your traffic right away, if the rumors are true, the NSA is archiving some or all encrypted traffic on the Internet so that they can break it later.

Forward Secrecy provides a great deal of armoring against this. RSA is the most questioned ingredient in the TLS sauce, and every few years the security experts announce that you should double your key size to maintain security. Without FS, if an adversary gets a hold of your private key and has captured the contents of previous SSL/TLS sessions, they can now decrypt all of them. Forward Secrecy uses RSA to authenticate the key exchange process, but uses a session key that is discarded after the session is complete. You're no longer using RSA to encrypt the session key, it can now be negotiated with Diffie-Hellman (or an elliptic-curve variant), which makes the captured session content useless to an adversary.

Also, I have OCD and like to make things perfect, and I majored in security.

Scope and goals
===============

What is included
----------------

I attempted to do all of the following:

* Mitigate all known protocol weaknesses, including BEAST, CRIME, POODLE, logjam and DROWN
* Provide only cipher suites offering [perfect forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy#Perfect_forward_secrecy)
* Score a "100" for cipher strength - this means disabling all suites weaker than 256 bits
  * **Update 2016-03-02:** Support for `ECDHE-RSA-AES128-GCM-SHA256` has been added temporarily until Google Chrome adds support for `ECDHE-RSA-AES256-GCM-SHA384`.
* Support almost all commonly used browsers

The result:

* **A+ score on Qualys SSL Report**
* 100 in key exchange category
* 90 in cipher strength category (due to `ECDHE-RSA-AES128-GCM-SHA256` being added for Chrome users)
* Your certificate score will depend on your private key size
* Protocol support has a 97.5/100 score, as TLS version 1.1 remains enabled

Out of scope
------------

The following items are out of scope for the copypasta config files included in this repo.

### Certificate chains and sending the root

GoDaddy (Datto's source of SSL certificates) likes to include the root certificate in their CA chain file. The root is the certificate that comes bundled in the browser. This results in a mostly harmless warning during the SSL Labs test. The only real harm is a waste of bandwidth.

Identifying the root is a somewhat tedious process. You'll need to use `openssl x509 -in ca-bundle.crt -noout -text`  to view all the certificates in the bundle, and visual-grep it until you identify the root. Once you find it, delete it from the bundle file.

### Key size

For the highest possible score you'll want a 4,096 bit private key. There is some performance impact to consider, but it's my opinion that 2,048 bit RSA will be declared not recommended within the next 10 years. If it's smaller, you will have to regenerate it and ask your CA for a re-key. GoDaddy does this for free, but other CAs may charge for this service.

### OpenSSL vulnerabilities

This configuration is really not going to be useful unless you patch OpenSSL.

On Ubuntu:

1. `grep -q security /etc/apt/sources.list || echo "Missing security repo"` - if this shows "Missing security repo", edit your `sources.list` appropriately.
1. `apt-get update`
1. `apt-get install openssl`

## Protocol support

Relatively recent versions of all the major browsers, particularly on mobile devices, may not support TLS v1.2, so TLS version 1.1 is enabled to maintain some level of compatibility. This is the reason the protocol support score is only 97.5/100 - Qualys awards you more points for supporting only the latest protocol versions, as their scoring is strictly based on security, not compatibility.

## FIPS 140-2

We don't care, and unless you're the government or a government contractor, you shouldn't either.

Setting up
==========

Apache
------

1. Put `apache-ssl-base.conf` in your `conf-available` directory or similar, and include it from your vhost definition.
1. Generate your own Diffie-Hellman parameters (`openssh dhparam -out dh4096.pem 4096`) or use ours. **IMPORTANT!** If you care about security in the long run, generating your own Diffie-Hellman parameters provides an additional level of security over just using ours. While we don't expect our 4,096 bit prime to become widely used (and therefore an NSA target), and we're pretty sure it will be a very long time before even the NSA can break a 4,096 bit prime, the safest option is just to generate your own. The more DH primes exist in the world, the more time the NSA needs to spend cracking them to be able to eavesdrop on everything. For further reading, see [weakdh.org](https://weakdh.org/).
1. Configure Diffie-Hellman:
  * Apache 2.4.7 or earlier: `cat dh4096.pem >> /path/to/certificate`
  * For Apache 2.4.8 or later, uncomment the `SSLOpenSSLConfCmd` directives in `apache-ssl-base.conf`. Update the path to `dh4096.pem` as necessary.
1. Reload Apache.

### HSTS

1. `a2enmod headers`
1. Add to `.htaccess` or your Apache vhost configuration: `Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"`

nginx
-----

1. Place `nginx-ssl-base.conf` in your `sites-enabled` directory or similar and include it inside your http definiton.
1. Add `ssl_trusted_certificate /etc/nginx/ssl-config/ca.pem;` to your server blocks, setting the path to your CA's certificate chain (including root!).
1. Reload nginx.

### HSTS

Add `add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";` to your server blocks.

License
=======

As the configuration files here consist only of a list of cipher suites and a few configuration directives, the contents of this repository are not eligible for copyright. Use it freely.

Of course, if this helps you out, some love on Twitter or in your page footer is always appreciated, but not required.

Author/Maintainer
=================

Dan Fuhry (<dfuhry@datto.com>)
