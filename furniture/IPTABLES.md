# System Defenses (with IPTABLES)

# IPTABLES 101

```sh
$ iptables -P INPUT DROP
$ iptables -A INPUT -p tcp -s 1.1.1.1 -j ACCEPT
```
> Assume 1.1.1.1 is your team members' IP addresses

This should be the very first commands you have done at the very start. (To prevent all access via whitelisting your own IP address).
More can also be done via whitelisting which applications/ports that only your IP address can access too.

```sh
$ iptables -A INPUT -p tcp -s 1.1.1.1 --dport 22 -j ACCEPT
```
> Allows only your IP address and only can access SSH service.

# POST-CONFIGURATION
This is when the Gameserver starts polling for services. You will need to allow ALL accesses to the SSH service and the HTTP service because these are the requirements.

```sh
$ iptables -P INPUT DROP
$ iptables -A INPUT -p tcp --dport 22 -j ACCEPT
$ iptables -A INPUT -p tcp --dport 80 -j ACCEPT
$ iptables -P OUTPUT DROP
$ iptables -A OUTPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT 
```
> These are some of the basic IPTABLES rules you should have. There are still more rules you can apply too, use your imagination!

For more information on IPTABLES, you can refer to this [article][iptables-tutorial].

[iptables-tutorial]: https://www.digitalocean.com/community/tutorials/iptables-essentials-common-firewall-rules-and-commands