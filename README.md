Ansible Role: Mautic
=========

`postedin.mautic` <https://galaxy.ansible.com/postedin/mautic/>

This role installs PHP 7.0 (using `geerlingguy.php`, so will keep in sync with that), git (`geerlingguy.git`) and composer (`geerlingguy.composer`).

If you want to use different PHP versions you will have to remove `geerlingguy.php` from `meta/main.yml` and then use <https://github.com/geerlingguy/ansible-role-php-versions/> or whatever PHP roles you want.

You can see a playbook using this role here: <https://github.com/postedin/ansible-playbook-mautic>

Requirements
------------

Only used on Ubuntu so might not work on others.

Role Variables
--------------

Default variables.

```yaml
mautic_version: "2.12.1"

mautic_destination: /opt/mautic
mautic_user: www-data
mautic_group: www-data

mautic_mysql_database: mautic
mautic_mysql_user: mautic
mautic_mysql_password: secret
```

Example variables we use on Ubuntu with nginx and MySQL locally.

```yaml
domain: marketing.postedin.com
mautic_destination: "/var/www/{{ domain }}"

# with a secret.yml which is git ignroed that contains `mautic_mysql_password`
```

Dependencies
------------

<https://github.com/geerlingguy/ansible-role-php>

<https://github.com/geerlingguy/ansible-role-git>

<https://github.com/geerlingguy/ansible-role-composer>

If using nginx you will need to set...

```yaml
php_enable_php_fpm: true
```

Example Playbook
----------------

Copied from <https://github.com/postedin/ansible-playbook-mautic.> Might be out of date so better to look at the actual playbook.

This playbook...

- sets domain var to be used everywhere
- has a secrets.yml for the mysql password
- adds the `www-data` group to the ubuntu user which we use for everything
- installs this role which `php_enable_php_fpm: true` and the `mautic_destination`
- installs mariadb and creates the database and user that mautic wants
- installs certbot and gets the letsencrypt certificates for the configured domain
- installs nginx with a vhost to redirect http -> https and the https one to serve mautic

```yaml
---
- hosts: web
  become: yes
  user: ubuntu
  vars:
    php_enable_php_fpm: true
    domain: marketing.postedin.com
    mautic_destination: "/var/www/{{ domain }}"
    mysql_packages:
      - mariadb-client
      - mariadb-server
      - python-mysqldb
  vars_files:
    - secret.yml
  pre_tasks:
    - name: add www-data group to ubuntu user
      user: name=ubuntu groups=www-data append=yes
  roles:
    - role: postedin.mautic
    - role: geerlingguy.mysql
      mysql_databases:
        - name: "{{ mautic_mysql_database }}"
      mysql_users:
        - name: "{{ mautic_mysql_user }}"
          password: "{{ mautic_mysql_password }}"
          host: "%"
          priv: "%.*:ALL"
    - role: geerlingguy.certbot
      certbot_admin_email: robbo@postedin.com
      certbot_create_if_missing: yes
      certbot_create_standalone_stop_services:
        - nginx
      certbot_certs:
        - domains:
            - "{{ domain }}"
    - role: geerlingguy.nginx
      nginx_vhosts:
        - listen: "80"
          server_name: "{{ domain }}"
          return: "301 https://{{ domain }}$request_uri"
          filename: "{{ domain }}.80.conf"
        - listen: "443 ssl http2"
          server_name: "{{ domain }}"
          root: "{{ mautic_destination }}"
          index: index.html index.htm index.php
          filename: "{{ domain }}.conf"
          extra_parameters: |
            rewrite ^/index.php/(.*) /$1  permanent;
            rewrite ^/(vendor|translations|build)/.* /index.php break;

            location / {
              try_files $uri /index.php$is_args$args;
            }

            location ~ ^/app/bundles/.*/Assets/ {
              allow all;
              access_log off;
            }

            location ~ ^/app/ { deny all; }

            location ~ ^/(addons|plugins)/.*/Assets/ {
              allow all;
              access_log off;
            }

            location ~ ^/(addons|plugins)/ { deny all; }

            location ~* ^/themes/(.*)\.php {
              deny all;
            }

            location = /favicon.ico {
              log_not_found off;
              access_log off;
            }

            location = /robots.txt  {
              access_log off;
              log_not_found off;
            }

            location ~* /(.*)\.(?:markdown|md|twig|yaml|yml|ht|htaccess|ini)$ {
              deny all;
              access_log off;
              log_not_found off;
            }

            location ~ /\. {
              deny all;
              access_log off;
              log_not_found off;
            }

            location ~* (Gruntfile|package|composer)\.(js|json)$ {
              deny all;
              access_log off;
              log_not_found off;
            }

            location ~ \.php$ {
              fastcgi_split_path_info ^(.+\.php)(/.+)$;
              fastcgi_pass 127.0.0.1:9000;
              fastcgi_index index.php;
              fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              include fastcgi_params;

              fastcgi_buffer_size 128k;
              fastcgi_buffers 256 16k;
              fastcgi_busy_buffers_size 256k;
              fastcgi_temp_file_write_size 256k;
            }

            ssl_certificate /etc/letsencrypt/live/{{ domain }}/fullchain.pem;
            ssl_certificate_key /etc/letsencrypt/live/{{ domain }}/privkey.pem;
            ssl_protocols TLSv1.1 TLSv1.2;
            ssl_ciphers HIGH:!aNULL:!MD5;

```

License
-------

MIT

Author Information
------------------

This role was created by [Postedin SpA](https://postedin.com)
