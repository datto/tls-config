# Datto's Secret TLS Sauce

As with most cloud service providers, security is Kind Of A Big Deal at Datto. With all the FUD going around about the NSA breaking SSL/TLS and stuff, it was a big priority for me to ensure that communications between Datto employees and devices and our mothership are secured.

The lowest hanging fruit for SSL/TLS experimentation was our mostly-internal LDAP/Kerberos account management interface, since only employees and a few distributors use it, and therefore we aren't as afraid of breaking older browsers.

If you just want to see results, [here's the test on sso.datto.net](https://www.ssllabs.com/ssltest/analyze.html?d=sso.datto.net&hideResults=on).

# Why?

There is [a lot of FUD](http://www.zdnet.com/article/has-the-nsa-broken-ssl-tls-aes/) going around on the Internet that the NSA and similar organizations have a vast reserve of zero-day attacks on TLS. Even if they can't decrypt your traffic right away, if the rumors are true, the NSA is archiving some or all encrypted traffic on the Internet so that they can break it later.

Forward Secrecy provides a great deal of armoring against this. RSA is the most questioned ingredient in the TLS sauce, and every few years the security experts announce that you should double your key size to maintain security. Without FS, if an adversary gets a hold of your private key and has captured the contents of previous SSL/TLS sessions, they can now decrypt all of them. Forward Secrecy uses RSA to authenticate the key exchange process, but uses a session key that is discarded after the session is complete. You're no longer using RSA to encrypt the session key, it can now be negotiated with Diffie-Hellman (or an elliptic-curve variant), which makes the captured session content useless to an adversary.

Also, I have OCD and like to make things perfect, and I majored in security.

# Scope and goals

## What is included

I attempted to do all of the following:

* Mitigate all known protocol weaknesses, including BEAST, CRIME, POODLE
* Provide [perfect forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy#Perfect_forward_secrecy)
* Score a "100" for cipher strength - this means disabling all suites weaker than 256 bits
* Support almost all commonly used browsers

The result:

* **A+ score on Qualys SSL Labs**
* 100 in key exchange and cipher strength categories
* Your certificate score will depend on your private key size=
* Protocol support has a 95/100 score, as TLS versions 1.0 and 1.1 remain enabled

## Out of scope

The following items are out of scope for the copypasta config files included in this repo.

### Certificate chains and sending the anchor

GoDaddy (Datto's source of SSL certificates) likes to include the anchor certificate in their CA chain file. The anchor is the certificate that comes bundled in the browser. This results in a mostly harmless warning during the SSL Labs test. The only real harm is a waste of bandwidth.

Identifying the anchor is a somewhat tedious process. You'll need to use `openssl x509 -in ca-bundle.crt -noout -text`  to view all the certificates in the bundle, and visual-grep it until you identify the anchor. Once you find it, delete it from the bundle file.

### Key size

Your private key should be absolutely no less than 4,096 bits. If it's smaller, you will have to regenerate it and ask your CA for a re-key. GoDaddy does this for free, but other CAs may charge for this service.

### HSTS

1. `a2enmod headers`
1. Add to `.htaccess` or your Apache vhost configuration: `Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"`

### OpenSSL vulnerabilities

This configuration is really not going to be useful unless you patch OpenSSL.

1. `grep -q security /etc/apt/sources.list || echo "Missing security repo"` - if this shows "Missing security repo", edit your `sources.list` appropriately.
1. `apt-get update`
1. `apt-get install openssl

## Protocol support

Relatively recent versions of all the major browsers may not support TLS v1.2, so TLS versions 1.0 and 1.1 are enabled to maintain some level of compatibility. This is the reason the protocol support score is only 95/100 - Qualys awards you more points for supporting only the latest protocol versions, as their scoring is strictly based on security, not compatibility.

## FIPS 140-2

We don't care, and unless you're the government or a government contractor, you shouldn't either.

# Setting up

## Apache

Put `apache-ssl-base.conf` in your `conf-available` directory or similar, and include it from your vhost definition. Reload Apache.

## nginx

Coming soon

# License

As the configuration files here consist only of a list of cipher suites and a few configuration directives, the contents of this repository are not eligible for copyright. Use it freely.

Of course, if this helps you out, some love on Twitter or in your page footer is always appreciated, but not required.

# Author/Maintainer

Dan Fuhry (<dfuhry@datto.com>)
