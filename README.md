# ![logo](https://raw.githubusercontent.com/jhermann/devpi-enterprisey/master/doc/static/logo-32.png) devpi-enterprisey

Augmentation of the awesome [devpi](http://doc.devpi.net/latest/) project
with ‘enterprisey’ requirements, configuration, and deployment.

![Apache 2.0 licensed](http://img.shields.io/badge/license-Apache_2.0-red.svg)

**Table of Contents**

- [Overview](#user-content-overview)
- [How-Tos](#user-content-how-tos)
    - [Installing the ‘devpi’ client into your home](#user-content-installing-the-devpi-client-into-your-home)
    - [Example for an index-per-team setup with shared global indexes](#user-content-example-for-an-index-per-team-setup-with-shared-global-indexes)
    - [Using `devpi` in Jenkins](#user-content-using-devpi-in-jenkins)
    - [SSL certificate handling](#user-content-ssl-certificate-handling)
- [Related Tickets](#user-content-related-tickets)
- [Related Projects](#user-content-related-projects)
- [References](#user-content-references)


## Overview

This project provides the following components:

* *Debian Packaging* – read the [docs](https://github.com/jhermann/devpi-enterprisey/tree/master/debianized-devpi) on how to build and install a `devpi` release as a DEB package, contained in a Python virtualenv.
* *Puppet Deployment* – see the example node definition in [site.pp](https://github.com/jhermann/devpi-puppet/blob/master/site.pp) on how to install devpi-server behind a NginX proxy, using the [devpi-puppet](https://github.com/jhermann/devpi-puppet) module.
* The [wiki](https://github.com/jhermann/devpi-enterprisey/wiki) contains some articles with details on different aspects of using `devpi` in an enterprise work environment.


## How-Tos

### Installing the ‘devpi’ client into your home

The following is an easy way to install the client on systems not derived from Debian (for Debian packaging, see above link).
It uses [pipsi](https://github.com/mitsuhiko/pipsi) to create a dedicated virtualenv for `devpi-client`.

```sh
grep '/.local/bin:' ~/.bash_aliases >/dev/null 2>&1 ||
    echo 'grep ":$HOME/.local/bin:" <<<":$PATH:" >/dev/null || export PATH="$HOME/.local/bin:$PATH"' >>~/.bash_aliases
source ~/.bash_aliases

which pipsi || curl https://raw.githubusercontent.com/mitsuhiko/pipsi/master/get-pipsi.py | python
which devpi || pipsi install devpi-client
devpi --version
```

The above command sequence is idempotent, i.e. you can repeat it as often as needed, in case of any problems.


### Example for an index-per-team setup with shared global indexes

The following shows how an index setup for a department with multiple teams can look like
– it tries to strike a balance between flexibility (i.e. independence and isolation of teams) and simplicity.

It's designed for the following requirements:

* Give each team a space to share experimental and stable releases, under their control.
* Provide a shared package pool, which consists of local reviewed releases, external releases not available on PyPI or patched locally, and finally the local PyPI proxy/cache.
* Releases that are not team-internal must be pushed through a QA gateway towards the shared package pool, i.e. `shared` is owned by the QA team.

![Sample Index Structure](https://raw.githubusercontent.com/jhermann/devpi-enterprisey/master/doc/static/repo-structure.png)

All team indexes eventually lead to `shared/stable`, a *virtual* index that is not supposed to hold any packages,
but only assemble other indexes into a common base for all teams.
The `shared/thirdparty-local` index is the only one that needs its `mirror_whitelist` maintained (or set to `*`);
`«user»/dev` is volatile, all others are not.

While the overall complexity of this setup can be staggering, it only ever reflects the complexity of your organization
(having a lot of teams). The *isolation* it provides actually helps to cope with that, since a single developer
only has to consider at most three of these indexes (his team's, and the virtual shared one).
There is no need to co-ordinate their use with other teams, and the QA gateway ensures the integrity of the shared pool.
In case you need to scale across several departments, you can simply apply the same pattern in a fractal way,
or set up discrete instances of `devpi-server`.


### Using `devpi` in Jenkins

First things first, every slave in your Jenkins farm should either get the `devpi` package installed,
or else an equivalent install of `devpi-client` (e.g. by `pipsi`).
Then add the `devpi_use` helper functions as described [here](https://github.com/jhermann/devpi-enterprisey/wiki/DevPIandJenkins)
into your Jenkins user's `~/.bashrc`.

Each job is responsible to select the proper index to use, by issuing the following command early on in its build step (take notice of the underscore in `devpi_use`):

```sh
devpi_use shared/stable # select the proper index for your project
# Now an alias for "devpi" is set, that uses "${WORKSPACE}/.devpi" as its clientdir;
# also sets PIP_INDEX_URL, so you can do «pip install …» without further thoughts.
```

The `shared/stable` index should be set as the user's default, so if you only need internally *released* or external packages, you can simply use pip without any extra considerations. You then don't need the above command, since `~/.pip/pip.conf` is used.
The final step of preparation is to install your project's requirements via a `requirements.txt` as usual

❢❢❢ Do NOT use devpi directly with `--set-cfg` or without a `--clientdir` pointing into your workspace on Jenkins, you'd otherwise create a hell of race conditions for yourself and others! Also, related config files shouldn't be writable anyway to prevent exactly that.


### SSL certificate handling

Accessing your `devpi` server over a secure connection is recommended, and also silences that pesky `pip` warning about insecure connections. Unless your certificate is issued by a well-known authority, you have to provide clients with the root certificates you're using, either self-signed or local PKI ones.

To query where your `devpi` client gets its CA certificates from, you can use the following scary-looking but harmless command:

```sh
$(head -n1 $(command which devpi) | tr -d '#!') -m requests.certs
```

In case you have your root CA certificates on a Debian system properly installed so that `update-ca-certificates` sees them, all you need is this (preferably added to a file in `/etc/profile.d`):

```sh
export PIP_CERT="/etc/ssl/certs/ca-certificates.crt"
export REQUESTS_CA_BUNDLE="$PIP_CERT"
```

Otherwise, let's say you have a bunch of local CA or self-signed certificates in `/usr/share/ca-certificates/acme.com`, then the following will make them accessible to both `devpi` and `pip`:

```sh
company=acme.com
mkdir -p ~/.local
export PIP_CERT="$HOME/.local/cacert.pem"
export REQUESTS_CA_BUNDLE="$PIP_CERT"
cat >"$PIP_CERT" $($(head -n1 $(command which devpi) | tr -d '#!') -m requests.certs) /usr/share/ca-certificates/$company/*.crt
```

In case your `devpi` server is a virtual host and shares its IP with a bunch of other domains, you also need SNI support for Python 2 (SNI = [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication)). To enable that, use your virtualenv's `pip` to install these packages:

```sh
pip install -U pip pyOpenSSL pyasn1 ndg-httpsclient
```

Note that we also update `pip` to the latest version here, since only recent ones can take advantage of the improved SSL packages.
You might also need to provide a HTTP `--index-url` or activate a `--proxy` to perform this initial bootstrap
– this is an example why it makes sense to keep a `devpi` server port for HTTP open for the time being.


## Related Tickets

* [Integrate LDAP-based authentication/authorization](https://bitbucket.org/hpk42/devpi/issue/138/integrate-ldap-based-authentication)
* [Prevent "pre" uploading / pushing into an index](https://bitbucket.org/hpk42/devpi/issue/137/prevent-pre-uploading-pushing-into-an)
* [Enable inheriting from external index (other than pypi.python.org)](https://bitbucket.org/hpk42/devpi/issue/12/enable-inheriting-from-external-index)
* [SSL certificate handling](https://bitbucket.org/hpk42/devpi/issue/64/ssl-certificate-handling)


## Related Projects

* [hpk42/devpi](https://bitbucket.org/hpk42/devpi)
* [devpi/devpi-findlinks](https://github.com/devpi/devpi-findlinks)
* [devpi/devpi-ldap](https://github.com/devpi/devpi-ldap) (pre-alpha as of 2014/09)
* [blue-yonder/devpi-builder](https://github.com/blue-yonder/devpi-builder) – Brandon, the devpi-builder, takes a `requirements.txt` and incrementally fills a `devpi` index with wheels of the listed Python packages.
* [blue-yonder/devpi-plumber](https://github.com/blue-yonder/devpi-plumber) – Mario, the devpi-plumber, helps to automate and test large `devpi` installations.
* [pypa/twine](https://github.com/pypa/twine) – Twine is a utility for interacting with PyPI.
* [pypa/bandersnatch](https://bitbucket.org/pypa/bandersnatch) – A PyPI mirror client implementing PEP 381, enabling you to go fully off-line with a complete copy of the packages hosted on the official PyPI index.


## References

* [packaging and testing with devpi and tox ](https://youtu.be/aNrrGf-uNUY) – Holger at EP 2014 in Berlin.
* [devpi: driving packaging and testing needs](http://youtu.be/84oOMBUUywI) – Holger's talk at PyCon DE 2013, an introduction to `devpi` and its philosophy (in English, shortly before the 1.2 release).
