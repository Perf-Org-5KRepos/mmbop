# mmbop

Dynamic DNS ([RFC 2136](https://tools.ietf.org/html/rfc2136)) provides a mechanism for updating entries in a zone file without needing to edit that file by hand and without needing to restart BIND. For Python users, the [dnspython](http://www.dnspython.org/) module provides a valuable foundation for creating dynamic DNS applications. However there are some things that dnspython and dynamic DNS cannot do - specifically, the creation of new authoratative zones.

BIND does come with a program, rndc, that can control various aspects of the name service, locally or remotely. One of the functions is adding a zone. However the command relies on the fact that the empty zone file already exists. So there is a need for a program to wrap rndc functionality and provide push button capability to add/remove zones. This is the purpose of mmbop.

## Getting Started

Clone the repository and you should be almost done. A (successful) attempt was made to only use modules from the standard library, so there are no Python dependencies when using the command-line (mmbop.py). For the API (mmbop_api.py), [Falcon](https://falconframework.org/) is used and required.

### Prerequisites

A working DNS environment would be useful. In *named.conf* (or associated configuration files), you will need a *controls* stanza that specifies the key that rndc is allowed to use, along with the acl defining the source (in our case, mmbop is run locally, so 127.0.0.1 - aka, localhost - is sufficient).

```
include "/etc/bind/rndc.key";

controls {
        inet * allow { localhost; } keys { "rndc-key"; };
};
```

If you are running BIND version 9.11.0 or later and wish to take advantage of the catalog zone feature (allowing you to automatically update secondary nameservers with the new zones you have created), the appropriate configuration is require on both the primary and secondary server(s).

Primary (for example sake, this has IP 10.1.1.1):
```
        catalog-zones {
                zone "catalog.zone";
        };

zone "catalog.zone" {
        type master;
        file "catalog.zone.db";
        allow-update { key rndc-key; };
};
```

Secondary:
```
        catalog-zones {
                zone "catalog.zone" default-masters { 10.1.1.1; };
        };

zone "catalog.zone" {
        type slave;
        file "/etc/bind/catalog.zone.db";
        masters { 10.1.1.1; };
};
```

See [this nice introduction](https://kb.isc.org/docs/aa-01401) to catalog zones for more information.

**Python and BIND version requirements**

The code was developed using Python 3.6.8 and BIND 9.14.3.
The code was successfully deployed using Python 3.7.4 and BIND 9.11.2rc2

Python 3.6+ is required as mmbop relies on a change in *subprocess* that accepts an *encoding* parameter. This matters for the catalog zone update.
BIND 9.11+ is required to support catalog zones.

### Installing

Assuming now you have at least a working primary DNS server that is accepting a local rndc connection, and you cloned this repository.

**Step 1 : Configure mmbop.ini**

Rename the example file to *mmbop.ini* and edit it so that the required values match your environment.

```
[RNDC]
server: 127.0.0.1
port: 953
keyfile: /etc/bind/rndc.key
path: /usr/sbin/rndc

[DNS]
dns1: ns1.example.com
dns2: ns2.example.com
serial: 1
owner: hostmaster.example.com
namedir: /etc/bind
nameown: bind
namegrp: bind
nameper: 0644

[MMBOP]
protect: example.com|example.net
require: .example.com|.example.net
options: also-notify { 10.10.10.1; };|allow-update { key "myddnskey"; };
catalog: catalog.example
Give the example
```
The .ini file comments explain each option, but the 2 least obvious ones (as they are specific to the mmbop application) are *protect* and *require*.

- protect: A list (entries separated by '|') of exact domain names that mmbop is not allowed to manage (add/remove). Typically used to protect the parent zone, if you are looking to make subdomains only.
- require: A list (entries separated by '|') of end of string matches. For mmbop to manage a domain, it must match at least one of these strings. Typically used to specify the subdomain (note preceeding '.' in example above).

**Step 2 : Verify by running mmbop for server status**

```
$ ./mmbop.py -s
version: BIND 9.14.4-Ubuntu (Stable Release) <id:ab4c496>
running on netdev-swtest: Linux x86_64 4.15.0-58-generic #64-Ubuntu SMP Tue Aug 6 11:12:41 UTC 2019
boot time: Wed, 21 Aug 2019 13:41:16 GMT
last configured: Wed, 21 Aug 2019 13:41:16 GMT
configuration file: /etc/bind/named.conf
CPUs found: 2
worker threads: 2
UDP listeners per interface: 2
number of zones: 32 (0 automatic)
debug level: 0
xfers running: 0
xfers deferred: 0
soa queries in progress: 0
query logging is OFF
recursive clients: 0/900/1000
tcp clients: 2/150
server is up and running
until finished
```
If you see output similar to the example above, congrats you have a working mmbop setup.
If this doesn't work, run with verbose logging (*-v*) for more details on the problem.

**Step 3 : Add a zone**

Given what mmbop is required to do to add a domain, it is necessary to run as a user with write permissions to the directory where the zone files are located.

You can run it as root (sudo), or - from root - you can sudo and run as the same user that owns the BIND service and files. By default, on Ubuntu this is the *bind* user, which is created when you install BIND.

```
# sudo -u bind /var/cache/bind/mmbop.py -c /var/cache/bind/mmbop.ini -a nina.example.com
Zone nina.example.com added

$ ./mmbop.py -z nina.example.com
name: nina.example.com
type: master
files: /etc/bind/nina.example.com.db
serial: 1
nodes: 1
last loaded: Thu, 22 Aug 2019 20:34:43 GMT
secure: no
dynamic: yes
frozen: no
reconfigurable via modzone: yes

```

**Step 4 : Remove a zone**

```
# sudo -u bind /var/cache/bind/mmbop.py -c /var/cache/bind/mmbop.ini -d nina.example.com
Zone nina.example.com deleted

$ ./mmbop.py -z nina.example.com
rndc: 'zonestatus' failed: not found
no matching zone 'nina.example.com' in any view

```

**Step 5 : Explore**

Run with *-h* or *--help* to see all of the available command-line options

```
$ ./mmbop.py -h
usage: mmbop.py [-h] [-v] [-c FILE] [-a ZONE | -d ZONE | -l | -s | -z ZONE]

mmbop manages BIND over Python

optional arguments:
  -h, --help                  show this help message and exit
  -v, --verbose               Enable verbose messages
  -c FILE, --config FILE      Location of mmbop config file
  -a ZONE, --add ZONE         Add specified zone
  -d ZONE, --delete ZONE      Delete specified zone
  -l, --list                  List all zones
  -s, --status                Show status of DNS server
  -z ZONE, --zonestatus ZONE  Show status of specified zone
```

Note that the list option, *-l*, will only show the domains that mmbop can manage (using same validation criteria as for adding/removing zones) - see above for the protect/require configuration.

## API

It may be desirable to provide the ability to add/remove zones, without wanting to give these users shell access to the primary DNS server. The script *mmbop.py* provides a simple REST API interface, which can be used with your favorite [WSGI](https://www.python.org/dev/peps/pep-3333/) capable web server to provide remote access to the application. There is a simple header-based authorization token solution for validating requests; for production use you will likely want something more robust (and/or handled directly through the web server controls).

**Step 1 : Configure mmbop_api.ini**

The only item in this configuration file is the SHA224 hashed hexidecimal digest of the token string that the client will provide in the web request.

To generate the hash, you can run python directly:

```
$ python3 -q
>>> import hashlib
>>> clear_token = 'mysupersecretkey'
>>> hash_token = hashlib.sha224(clear_token.encode()).hexdigest()
>>> print(hash_token)
fb096d8fa48b12cf2adec03e5e5d03fb231bb87674f8d8dbf137f05c
```
Added to the *mmbop_api.ini* file:

```
[DEFAULT]
token:fb096d8fa48b12cf2adec03e5e5d03fb231bb87674f8d8dbf137f05c
```
Given this, for a request to be valid the client must send *mysupersecretkey* as the value in the *Authorization* field of the header. See the examples below.

**Step 2 : Start service**

For the purpose of providing an example, gunicorn will be directly used. By default this starts the service listening locally on port 8000.

See [this guide](https://www.digitalocean.com/community/tutorials/how-to-deploy-falcon-web-applications-with-gunicorn-and-nginx-on-ubuntu-16-04) for using gunicorn with nginx.

For production, you should use https to keep the auth token secure.

```
$ sudo gunicorn mmbop_api:APP
[2019-08-22 15:35:53 -0400] [16586] [INFO] Starting gunicorn 19.9.0
[2019-08-22 15:35:53 -0400] [16586] [INFO] Listening at: http://127.0.0.1:8000 (16586)
[2019-08-22 15:35:53 -0400] [16586] [INFO] Using worker: sync
[2019-08-22 15:35:53 -0400] [16589] [INFO] Booting worker with pid: 16589
```

Example 1: Listing all zones:

```
$ curl -i -H "Content-Type: application/json" -H "Authorization: mysupersecretkey" http://127.0.0.1:8000/zonelist
HTTP/1.1 200 OK
Server: gunicorn/19.9.0
Date: Thu, 22 Aug 2019 19:42:08 GMT
Connection: close
content-length: 41
content-type: application/json
scott.example.com sarah.example.net
```
The use of *-i* in curl is just to show the response code, not required for obtaining a result

Example 2: Adding an invalid zone (does not meet the protect/require standards):

```
$ curl -i -X POST -H "Content-Type: application/json" -H "Authorization: mysupersecretkey" -d '{"domain":"bogus.company.com"}' http://127.0.0.1:8000/modify
HTTP/1.1 400 Bad Request
Server: gunicorn/19.9.0
Date: Thu, 22 Aug 2019 20:14:45 GMT
Connection: close
content-length: 43
content-type: application/json

Not a valid zone name. Check configuration.
```

**Current list of API functions and their allowed methods:**

- /status [GET]
- /modify [POST for add, DELETE for delete]
- /zonelist [GET]
- /zoneinfo/{domain} [GET]

## Built With

* [VIM](https://www.vim.org/) - venerable and more than capable
* [Pylint](https://www.pylint.org/) - keeping my code somewhat under control and consistent

## Authors

* **Scott Strattner ** - [IBM Github](https://github.ibm.com/sstrattn)

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Acknowledgments

* Without an internet search engine none of this would be possible
* Thanks to dnspython I was able to reverse engineer the weird 'wire format' required for catalog zone entries
* [This](https://gist.github.com/PurpleBooth/109311bb0361f32d87a2) nice README template