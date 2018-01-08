# Challenge (Troubleshoot)

# Location
Located at `troubleshoot.php` after sign-in

# Vulnerability
1. Command Injection
2. Local File Inclusion

The code section in question
```html
<?php require_once("./headnav.php"); ?>
    <div class="container">
      <h1>Troubleshoot<?php if(isset($_POST['p'])) { echo ' - '.strtoupper($_POST['p']);} ?></h1>
      
      <form action="troubleshoot.php" method="POST">
          <input type="radio" name="p" value="ping.php"/> Ping<br/>
          <input type="radio" name="p" value="traceroute.php"/> Traceroute<br/>
          <input type="hidden" name="ip" value="8.8.8.8"/>
          <input type="submit" value="Submit"/>
      </form>
      <br/>
      <?php
        if (isset($_POST['p']) && isset($_POST['ip'])){
          echo '<pre>';
          include $_POST['p'];
          echo '</pre>';
        }
      ?>
```

## 1. Command Injection
The command injection vulnerability is due to the fact that `ip` are not filtered/sanitized in `troubleshoot.php`, `ping.php` and `traceroute.php`. And when you look into `ping.php` as an example below.

```php
<?php
system('ping -c 1 -s 1 '.$_POST['ip']);
?>
```

It is a simple line that runs the command `ping` with `ip` as a variable. As you can see, it is unsanitized. Therefore, you can exploit this by using the string below to exploit it and run any system commands.

```
ip=8.8.8.8; cat /etc/passwd;
```
> In this example, we use it to read the contents of /etc/passwd

You can exploit this via the use of **Burp suite** or **curl** or any tool that can send POST requests.

With the ability to read any contents your privilege allows you, you can read the **bonus flag**, **admin flag**, **document root flag**, **access log flag** and potentially **database flag**. (only lacking in the **root flag**)

This vulnerability can also allow you to either download a reverse php shell to the system for you to access or run a command that gives you back a reverse shell (eg. netcat, bash).

## 2. Local File Inclusion
The local file inclusion vulnerability is due to the fact that `p` are not filtered/sanitized in `troubleshoot.php`. And whatever file is being included with the code section below.

```php
      <?php
        if (isset($_POST['p']) && isset($_POST['ip'])){
          echo '<pre>';
          include $_POST['p'];
          echo '</pre>';
        }
      ?>
```

This local file inclusion vulnerability can allow you to read the flag files too. But because `/var/log/apache2/offsecstore_access.log`'s permission allows read and write to `www-data`. It is possible that you are able to write a PHP line to run system command to this file. You can do that by adding PHP line to the HTTP user-agent header when you are sending the HTTP request to the web server. The web server will store this in `/var/log/apache2/offsecstore_access.log`.

And when you use PHP include to access this file, the PHP line will be treated as an actual PHP code and be executed.

Example of the HTTP request can be like below.

```
GET / HTTP/1.0
User-Agent: <?php system('cat /etc/passwd'); ?>
Cookie: ...
```

# Solution
These two vulnerabilities can be resolved below.

The command injection vulnerability can be resolved if sanitization is added to `ip` and make sure nothing other than IP addresses can be the content of the variable.

The local file inclusion vulnerability can also be resolved with a simple method by changing it to allow only PHP files with the extension.

A simple example can be seen below.

In `troubleshoot.php`, change these to the filenames.
```html
<input type="radio" name="p" value="ping"/> Ping<br/>
<input type="radio" name="p" value="traceroute"/> Traceroute<br/>
```

And the "include" line of code.
```php
include $_POST['p'].'php';
```

This should prevent local file inclusion attack from being easily executed.