# Challenge (Bonus)
This challenge is meant as a bonus because it is easy and even after fixing it, it is still possible to obtain its flag legitimately.

# Location
Located at `bonus.php`

# Vulnerability
1. Weak Typing

The vulnerability is because the web application uses `==` instead `===` for performing matching condition. `===` should have be used because it enforces strict type matching while `==` will try to convert to a type that it "thinks" is correct.

Let's refer to chunk of code that does the matching.

```php
<?php
        if (isset($_POST['secret'])) {
          $s = $_POST['secret'];
          if (md5($s) == '0e689360761328193896196672040289'){
              echo file_get_contents('./bonus/.bonus_flag');
          }
        }
?>
```
> Note: What does `0e689360761328193896196672040289` looks similar to you? Answer is integer that has exponent. And therefore, this is what PHP will try to convert to: Integer.

To solve this, you simply need to enter `240610708` into the input box and boom, you got the flag. Reference from [Whitehat Security](https://www.whitehatsec.com/blog/magic-hashes/)

# Solution
As mentioned above, you can fix this by changing `==` to `===`.

```php
if (md5($s) === '0e689360761328193896196672040289')
```
> Note: Due to the requirement of the competition, you are not allowed to the hash or the displaying of the flag.

After this vulnerability is fixed, **IF** the attacker can still somehow guess original string or reverse the MD5 hash, this flag will still be displayed. This is intentional as original writer would like to give some **bonus** points if the attacker is willingly to put in extra effort to guess/reverse it.