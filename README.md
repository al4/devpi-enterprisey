# ![logo](https://raw.githubusercontent.com/jhermann/devpi-enterprisey/master/doc/static/logo-32.png) devpi-enterprisey

Augmentation of the awesome [devpi](http://doc.devpi.net/latest/) project
with ‘enterprisey’ requirements, configuration, and deployment.

This project provides the following components:

* *Debian Packaging* – read the [docs](https://github.com/jhermann/devpi-enterprisey/tree/master/debianized-devpi) on how to build and install a `devpi` release as a DEB package, contained in a Python virtualenv.
* *Puppet Deployment* - see the example node definition in [puppet](https://github.com/jhermann/devpi-enterprisey/tree/master/puppet) on how to install devpi-server behind a NginX proxy, using the [devpi-puppet](https://github.com/jhermann/devpi-puppet) module.


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


## Related Tickets

* [Integrate LDAP-based authentication/authorization](https://bitbucket.org/hpk42/devpi/issue/138/integrate-ldap-based-authentication)
* [Prevent "pre" uploading / pushing into an index](https://bitbucket.org/hpk42/devpi/issue/137/prevent-pre-uploading-pushing-into-an)
* [Enable inheriting from external index (other than pypi.python.org)](https://bitbucket.org/hpk42/devpi/issue/12/enable-inheriting-from-external-index)


## References

* [hpk42/devpi](https://bitbucket.org/hpk42/devpi)
* [devpi/devpi-findlinks](https://github.com/devpi/devpi-findlinks)
* [devpi/devpi-ldap](https://github.com/devpi/devpi-ldap) (pre-alpha as of 2014/09)
