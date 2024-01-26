## Apache2 & Wordpress Multisite setup in Debian 12

This setup doesn't make use of wordpress's `multisite` feature. While it is still enabled by default, the only difference is that we give each site its own wordpress configuration, instead of writing it all into one. So we don't need to share a database or configuration file between sites.

It's a small difference.

The contents of these configurations, as well as the permissions are taken from the debian wiki or the debian's documentation for wordpress. Please check them for further explanation.

### Documentation
```
/usr/share/doc/wordpress/README.Debian
https://wiki.debian.org/WordPress


http://codex.wordpress.org/Editing_wp-config.php
```

### scripts & samples
```
/usr/share/doc/wordpress/examples/setup-mysql.gz
/usr/share/doc/wordpress/examples/apache.conf

/usr/share/doc/wordpress/examples/config-default.php
/usr/share/wordpress/wp-config-sample.php

```
there might be more scripts in that folder.

### Process

#### Install necessary packages
```
sudo apt update
sudo apt install wordpress curl apache2 mariadb-server
```

#### (Optional) Install `iputils-ping`
 for `setup-mysql` script to work. This assumes your site is ready to go online.

```
sudo apt install iputils-ping
```

#### run helper scripts

run `mysql_secure_installation` or `/usr/share/doc/wordpress/examples/setup-mysql`. The latter might conflict with the former. Neither is necessary as that shit can be done manually.

Be sure to have a password manager ready, for database and wordpress passwords.

```
sudo mysql_secure_installation
```

#### Create some configs

Depending on the routes taken, some of this might be taken care of already. Here's an approach that should work out of the box for a multisite install, using a single wordpress core installation, with independent `wp-content` folders.

- create **apache .conf** file

```
sudo vim /etc/apache2/sites-available/<site>.conf
```

For example, `example1.conf`

and fill it with:

```
## Virtual host VirtualDocumentRoot

    NameVirtualHost *:80

    <VirtualHost *:80>
    UseCanonicalName Off
    VirtualDocumentRoot /usr/share/wordpress
    Options All

    # wp-content in /srv/www/wp-content/$0
    RewriteEngine On
    RewriteRule ^/wp-content/(.*)$ /srv/www/wp-content/${HTTP_HOST}/$1
    </VirtualHost>

```

create **wordpress config** for the site

```
sudo vim /etc/wordpress/config-example1.test.php
```

(It's important that it starts with `config-`)

Now fill it with this content (note, that if the site is to go live, this is where a lot of hardening can be done).
This should also correspond with the setup you did with the script earlier, as it referenes the mysql/mariadb database.

```
<?php
define('DB_NAME', 'wordpress_ex1');
define('DB_USER', 'wordpress');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
define('WP_CONTENT_DIR', '/srv/www/wp-content/example1.test');
?>
```

Create the `wp-content` directory for each site, and set the right permissions and ownership for a standard site. That means we can manage the default themes and plugins via wordpress, and generally do more from within. The core files of wordpress are unchanged.

```
sudo mkdir -p /srv/www/wp-content/example1.test
sudo cp -r /var/lib/wordpress/wp-content/* /srv/www/wp-content/example1.test

# remove symlink and create real copy of plugins
sudo rm -rf /srv/www/wp-content/example1.test/plugins/*
sudo cp -r /usr/share/wordpress/wp-content/plugins/* /srv/www/wp-content/example1.test/plugins/*

# remove symlink and create real copy of themes
sudo rm -rf /srv/www/wp-content/example1.test/themes/*
sudo cp -r /usr/share/wordpress/wp-content/themes/* /srv/www/wp-content/example1.test/themes/*

sudo chown -R www-data:www-data /srv/www/wp-content/example1.test
```

#### enable that shit and restart apache

```
sudo a2enmod rewrite && sudo a2enmod vhost_alias && sudo systemctl apache2 restart
```

- disable the default conf and enable your conf
```
sudo a2dissite 000-default
sudo a2ensite <site>
```

`<site>` is in our example `example1`

And done. Repeat the necessary steps for each additional site.

Visit the site at example1.test. Might also want to try example1.test/admin example1.test/wp example1.test/wp-admin

### notes

If the default theme `twentytwentythree` doesn't load and it's not in the folder, just reinstall it.

```
sudo apt-get --reinstall install wordpress-theme-twentytwentythree
```

There are very few safe Top-Level-Domains (TLDs) for testing.
Only use domains you own, or these, specifically for private/testing reserved TLDs by RFC 2606 & 6761. You can still use subdomains.

```
.test
.example
.invalid
.localhost
```
