# postfix ![status ready](https://img.shields.io/badge/status-ready-brightgreen.svg)

#### Table of Contents

1. [Overview](#overview)
2. [Module Description](#module-description)
3. [Setup](#setup)
    * [What postfix affects](#what-postfix-affects)
    * [Setup requirements](#setup-requirements)
    * [Beginning with postfix](#beginning-with-postfix)
4. [Usage](#usage)
5. [Reference](#reference)
5. [Limitations](#limitations)
6. [Development](#development)
    * [Contributing](#contributing)

## Overview

postfix configuration management

## Module Description

postfix setup and configuration, can be configured to act like a simple mail relay or a multidomain mailserver

## Setup

### What postfix affects

* /etc/postfix/main.cf
* package management
* service management
* purges packages for other MTA on CentOS and switches to postfix on Ubunut 14.04

### Setup Requirements

This module **requires pluginsync** enabled and **eyp-dovecot** (it is required to be able to setup IMAP for the mailserver: **postfix::vmail**)

### Beginning with postfix

#### basic setup:

```puppet
class { 'postfix':
  inetinterfaces => 'localhost',
}
```

#### mail relay:

```puppet
class { 'postfix':
  inetinterfaces => 'all',
  relayhost      => '1.2.3.4',
  mynetworks     => [ '127.0.0.1/32', '1.1.1.1/32' ],
}
```

#### multidomain mail server

```
class { 'postfix': }

class { 'postfix::vmail': }

postfix::vmail::alias { 'example@systemadmin.es':
  aliasto => [ 'exemple@systemadmin.es' ],
}

postfix::vmail::account { 'example@systemadmin.es':
  accountname => 'example',
  domain      => 'systemadmin.es',
  password    => 'secretpassw0rd',
}

postfix::vmail::account { 'silvia@systemadmin.es':
  accountname => 'silvia',
  domain      => 'systemadmin.es',
  password    => 'secretpassw0rd2',
}

postfix::vmail::account { 'marc@systemadmin.es':
  accountname => 'marc',
  domain      => 'systemadmin.es',
  password    => 'secretpassw0rd3',
}
```

## Usage

This module can be used to configure postfix to relay mails to another server or to have virtual mailboxes (multidomain/multiaccount).

To setup **opportunistic TLS with custom certificates**:

```puppet
	class { 'postfix':
		opportunistictls => true,
		tlscert          => 'puppet:///openldap/masterauth/ldap-master-01.crt',
		tlspk            => 'puppet:///openldap/masterauth/ldap-master-01.key.pem',
	}
```

To setup **opportunistic TLS with selfsigned certificate**:

```puppet
	class { 'postfix':
		opportunistictls  => true,
		subjectselfsigned => '/C=ES/ST=Barcelona/L=Barcelona/O=systemadmin.es/CN=systemadmin.es',
		generatecert      => true,
	}
```

Mailserver with contentfilter (amavis)

```puppet
# Mailserver
class { 'postfix': }

class { 'postfix::vmail': }

postfix::vmail::account { 'merda@merda.com':
  accountname => 'merda',
  domain      => 'merda.com',
  password    => 'putamerda',
}

class { 'postfix::contentfilter':
}
```

multiple smtp outbound instances:

```
postfix::instance { 'out_domain1':
  type    => 'unix',
  chroot  => 'n',
  command => 'smtp',
  opts    => { 'smtp_bind_address' => '1.1.1.1',
               'smtp_helo_name' => 'systemadmin.es',
               'syslog_name' => 'postfix-systemadmin.es',
             }
}

postfix::instance { 'out_domain2':
  type    => 'unix',
  chroot  => 'n',
  command => 'smtp',
  opts    => { 'smtp_bind_address' => '1.2.2.2',
               'smtp_helo_name' => 'sysadmins.es',
               'syslog_name' => 'postfix-sysadmins.es',
             }
}
```

blackhole domain or account (to be able to blackhole a domain it requires **postfix::vmail**):

```puppet
postfix::alias { 'blackhole':
  to => '/dev/null',
}

postfix::vmail::alias { '@blackhole.com':
  aliasto => [ 'blackhole@' ],
}
```

log example:

```
# echo a | mail -s caca blackhole@

Nov 29 12:33:03 ldapm postfix/pickup[16927]: 51876A105B: uid=0 from=<root>
Nov 29 12:33:03 ldapm postfix/cleanup[16995]: 51876A105B: message-id=<20161129113303.51876A105B@ldapm>
Nov 29 12:33:03 ldapm postfix/qmgr[16928]: 51876A105B: from=<root@vm.vm>, size=384, nrcpt=1 (queue active)
Nov 29 12:33:03 ldapm postfix/local[16997]: 51876A105B: to=<blackhole@ldapm>, orig_to=<blackhole@>, relay=local, delay=0.09, delays=0.07/0.03/0/0, dsn=2.0.0, status=sent (delivered to file: /dev/null)
Nov 29 12:33:03 ldapm postfix/qmgr[16928]: 51876A105B: removed

# echo a | mail -s caca dsadadas@blackhole.com

Nov 29 12:33:10 ldapm postfix/pickup[16927]: 70BA8A105B: uid=0 from=<root>
Nov 29 12:33:10 ldapm postfix/cleanup[16995]: 70BA8A105B: message-id=<20161129113310.70BA8A105B@ldapm>
Nov 29 12:33:10 ldapm postfix/qmgr[16928]: 70BA8A105B: from=<root@vm.vm>, size=396, nrcpt=1 (queue active)
Nov 29 12:33:10 ldapm postfix/local[16997]: 70BA8A105B: to=<blackhole@ldapm>, orig_to=<dsadadas@blackhole.com>, relay=local, delay=0.03, delays=0.02/0/0/0, dsn=2.0.0, status=sent (delivered to file: /dev/null)
Nov 29 12:33:10 ldapm postfix/qmgr[16928]: 70BA8A105B: removed
```

multiple inbound email instances:

```puppet
class { 'postfix':
  inetinterfaces    => 'all',
  mynetworks        => [ '127.0.0.1/32' ],
  myhostname        => 'smtp3.systemadmin.es',
  smtpdbanner       => 'smtp3.systemadmin.es ESMTP',
  opportunistictls  => true,
  subjectselfsigned => '/C=UK/ST=Shropshire/L=Telford/O=systemadmin/CN=smtp3.systemadmin.es',
  generatecert      => true,
  syslog_name       => 'private',
}

class { 'postfix::vmail': }

postfix::vmail::account { 'systemadmin@systemadmin.es':
  accountname => 'systemadmin',
  domain      => 'systemadmin.com',
  password    => 'systemadmin_secret_passw0rd',
}

postfix::instance { '0.0.0.0:2525':
  type    => 'inet',
  private => 'n',
  chroot  => 'n',
  command => 'smtpd',
  opts    => {
              'content_filter'               => '',
              'smtpd_helo_restrictions'      => '',
              'smtpd_sender_restrictions'    => '',
              'smtpd_recipient_restrictions' => 'permit_mynetworks,reject',
              'mynetworks'                   => '127.0.0.0/8,10.0.2.15/32',
              'receive_override_options'     => 'no_header_body_checks',
              'smtpd_helo_required'          => 'no',
              'smtpd_client_restrictions'    => '',
              'smtpd_restriction_classes'    => '',
              'disable_vrfy_command'         => 'no',
              #'strict_rfc821_envelopes'      => 'yes',
              'smtpd_sasl_auth_enable'       => 'no',
              'syslog_name'									 => 'public',
            },
  order   => '99',
}
```

## Reference

### postfix

Most variables are standard postfix variables, please refer to postfix documentation:
 * append_dot_mydomain
 * biff
 * inetinterfaces
 * ipv6
 * mail_spool_directory
 * mydestination
 * mydomain
 * myhostname
 * mynetworks
 * myorigin
 * readme_directory
 * recipient_delimiter
 * relayhost
 * smtp_fallback_relay
 * smtpdbanner
 * install_mailclient
 * default_process_limit
 * smtpd_client_connection_count_limit
 * smtpd_client_connection_rate_limit
 * in_flow_delay
 * setgid_group
 * (...)

* **install_mailclient**: controls if a mail client should be installed (default: true)

#### SSL certificates:
* **opportunistictls**: controls Opportunistic TLS (default: false)
* **generatecert**: controls if a selfsigned certificate is generated for this postfix instance (default: true)
* **tlscert**: source cert file - **generatecert** must be false
* **tlspk**: source private key - **generatecert** must be false
* **subjectselfsigned** subject for a selfsigned certificate - **generatecert** must be true. example: '/C=RC/ST=Barcelona/L=Barcelona/O=systemadmin.es/CN=systemadmin.es',

### postfix::transport

bounce a specific domain:

```puppet
postfix::transport { 'example.com':
  error => 'email to this domain is not allowed',
}
```

SMTP route:

```puppet
postfix::transport { 'example.com':
  nexthop => '1.1.1.1',
}
```

### postfix::vmail

* **mailbox_base**: (default: /var/vmail)
* **setup_dovecot**: (default: true)
* **smtpd_recipient_restrictions** (default: permit_inet_interfaces,permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination)
* **smtpd_relay_restrictions** (default: permit_inet_interfaces,permit_mynetworks,permit_sasl_authenticated,reject_unauth_destination)

### postfix::vmail::acount

```puppet
postfix::vmail::account { 'silvia@systemadmin.es':
  accountname => 'silvia',
  domain      => 'systemadmin.es',
  password    => 'secretpassw0rd2',
}
```

### postfix::vmail::alias

```puppet
postfix::vmail::alias { 'example@systemadmin.es':
  aliasto => [ 'exemple@systemadmin.es' ],
}
```

## Limitations

Tested on:
* CentOS 5
* CentOS 6
* CentOS 7
* Ubuntu 14.04
* Ubuntu 16.04
* SLES 11 SP3

## Development

We are pushing to have acceptance testing in place, so any new feature should
have some test to check both presence and absence of any feature

### TODO

* improve documentation (multidoamin mailserver is not yet covered)
* SQLite support (was added with Postfix version 2.8)
* add requires for postmap operations and rewrite it to use ${postfix::params::baseconf}

### Contributing

1. Fork it using the development fork: [jordiprats/eyp-systemd](https://github.com/jordiprats/eyp-systemd)
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
