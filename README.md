# ansible-php

Setups PHP from [ondrej/php](https://launchpad.net/~ondrej/+archive/ubuntu/php) PPA

## Example playboot

```yaml
---
- hosts: myserver
  user: root
  roles:
    - role: sunfoxcz.php
      php_version: 7.2
      php_modules:
        - bcmath
        - bz2
        - cli
        - curl
        - fpm
        - gd
        - imap
        - intl
        - json
        - mbstring
        - mysql
        - opcache
        - readline
        - soap
        - sqlite3
        - xml
        - zip
      php_ini_settings:
        memory_limit: 256M
        upload_max_filesize: 256M
        post_max_size: 256M
        error_reporting: "E_ALL & ~E_NOTICE & ~E_STRICT & ~E_DEPRECATED"
        expose_php: Off
        user_ini.filename: ""
        session.cookie_lifetime: 31536000
        session.gc_maxlifetime: 31536000
        date.timezone: Europe/Prague
      php_pools:
        myapp:
          user: myapp
          home: /www/myapp
          port: 9001
          max_children: 120
          start_servers: 20
          min_spare_servers: 10
          max_spare_servers: 20
    - role: sunfoxcz.php
      php_version: 5.6
      php_modules:
        - bcmath
        - bz2
        - cli
        - curl
        - fpm
        - gd
        - imap
        - intl
        - json
        - mbstring
        - mysql
        - opcache
        - readline
        - soap
        - sqlite3
        - xml
        - zip
      php_ini_settings:
        memory_limit: 256M
        upload_max_filesize: 256M
        post_max_size: 256M
        error_reporting: "E_ALL & ~E_NOTICE & ~E_STRICT & ~E_DEPRECATED"
        expose_php: Off
        user_ini.filename: ""
        session.cookie_lifetime: 31536000
        session.gc_maxlifetime: 31536000
        date.timezone: Europe/Prague
      php_pools:
        legacyapp:
          user: myapp
          home: /www/myapp
          port: 9000
          max_children: 120
          start_servers: 20
          min_spare_servers: 10
          max_spare_servers: 20
```

## License

Licensed under MIT license. See [LICENSE](LICENSE.md) for details.
