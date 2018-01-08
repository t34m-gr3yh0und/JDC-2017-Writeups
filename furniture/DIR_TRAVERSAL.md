# Challenge (Directory Traversal)
This is the easiest to exploit and can be used to obtain multiple flags very easily. Therefore, this should have been your top priority to secure when you get access to your web application.

# Location
Located at URL

# Vulnerability
1. Directory Traversal

You can easily obtain the **admin flag**, the **bonus flag** and the **document root flag** by visiting the page below.

```
http://1.1.1.1/admin/.admin_flag
http://1.1.1.1/bonus/.bonus_flag
http://1.1.1.1/.docroot_flag
```
> Assume 1.1.1.1 is the web application's IP address

# Solution
1. Create `.htaccess` at the Document Root of the web application
2. Add the lines below into the `.htaccess` file
```
<FilesMatch "flag$">
    deny from all
</FilesMatch
```
> Note: This is just a simple way to deny direct access to the files you want to hide.
3. Restart Apache service.