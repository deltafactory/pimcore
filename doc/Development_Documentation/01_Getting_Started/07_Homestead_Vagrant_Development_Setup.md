# Homestead/Vagrant Development Setup

To avoid having to develop on your local machine (and having to install a lot of dependencies on your machine) or on a remote
server, you can use [Vagrant](https://www.vagrantup.com/) to build a virtualized development environment. [Homestead](https://laravel.com/docs/master/homestead)
provides a simple configuration based Vagrant setup which can be used to quickly provision development environments. Your
code/project will be mounted to the virtual environment which means you can develop locally without having to take care
of syncing any changes to the virtual machine.

First of all, please read the installation docs for Homestead and make sure vagrant and a virtualization platform as VirtualBox
is properly installed. 

Assuming we work in a `~/work` directory on a *nix host (see Homestead docs for Windows details) we clone both the Pimcore
and the Homestead repository and set up Homestead to serve Pimcore:

```
$ mkdir ~/work
$ cd ~/work
$ git clone https://github.com/laravel/homestead.git
$ git clone https://github.com/pimcore/pimcore.git
$ cd homestead
$ git checkout 6.0.4 # needs to be something after 6.0.3 with added Pimcore support
$ bash init.sh
```

## Configure Homestead

This bootstraps the homestead installation. The `init.sh` creates 2 files `Homestead.yaml` and
`after.sh` which can be used to customize the setup. Let's map the pimcore checkout to a host 
in homestead by editing `Homestead.yaml`:

```yaml
---
# IP can be anything, but make sure it doesn't interfere with other interfaces
ip: "10.0.85.15"
memory: 2048
cpus: 1
provider: virtualbox
name: homestead
mariadb: true

authorize: ~/.ssh/id_rsa.pub
keys:
    - ~/.ssh/id_rsa

folders:
    - map: ~/work/pimcore
      to: /home/vagrant/pimcore

sites:
    - map: pimcore.app
      to: /home/vagrant/pimcore/web
      type: pimcore

databases:
    - pimcore
```

## Updating `/etc/hosts`

To have your machine automatically resolve the host `pimcore.app` you can install the `vagrant-hostsupdater` plugin
which takes care of mapping Vagrant IPs to hostnames by editing your `/etc/hosts` file:

```
$ vagrant plugin install vagrant-hostsupdater
```

If you can't or don't want to install the plugin you need to update your `/etc/hosts` file manually.

## Installing/Provisioning Pimcore

The second file mentioned above is the `after.sh` which you can use to add custom provisioning
logic. You can use the following template to run the CLI installer for all your desired installations. This
will be executed during provisioning (first start of a virtual machine or when called with `--provision`). 

```bash
#!/bin/bash

set -e

# installs pimcore
# expects the following
#   - first argument is the site and DB name (site and DB name must match)
#   - second argument is the install profile to use (defaults to empty)
#   - site is mapped to /home/vagrant/<site name>
install_pimcore() {
    cd ~/$1

    # system was already installled - skip installation
    if [ -e var/config/system.php ]; then
        >&2 echo "var/config/system.php was found in $1...skipping installation"
        return
    fi

    # prepare DB - change character set to utf8mb4
    mysql -e "ALTER DATABASE \`$1\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"

    # install composer dependencies
    composer install

    # install pimcore
    bin/install --symlink --profile ${2:-empty} --no-interaction \
        --admin-username admin --admin-password admin \
        --mysql-username homestead --mysql-password secret --mysql-database $1
}

# install pimcore
export PIMCORE_ENVIRONMENT=dev
install_pimcore pimcore demo-basic-twig

# install further instances
# install_pimcore pimcore-test basic-cms
# ...
```

## Starting the machine

As you have every set up, you can try to start your development environment:

```
$ cd ~/work/homestead
$ vagrant up
```

This will take a couple of minutes on the first start as it needs to download, import and provision
your machine. If everything went well you should be able to open [https://pimcore.app](https://pimcore.app)
in your browser and see a working Pimcore installation.
