# SSH into Vagrant

> Setup SSH and a Vagrant guest

## Terms

* host - host machine
* guest - the "host" we're creating using Vagrant and VirtualBox

## Networking

By default Vagrant creates one interface eth0 with a private ip address. It talks to outside world using Network Address Translation. It can't be reached from the host.

```
vagrant ssh -c 'ifconfig eth0' | grep -B 1 'inet addr'
eth0      Link encap:Ethernet  HWaddr 08:00:27:ee:1d:f7
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0

ping 10.0.2.15
PING 10.0.2.15 (10.0.2.15): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
^C
--- 10.0.2.15 ping statistics ---
3 packets transmitted, 0 packets received, 100.0% packet loss
```

Why can't it be reached from the outside?

The host machine doesn't have a way inside. There is no route to the guest

```
netstat -rn | grep 10.0.2
echo $?
1
```

How do we fix this this?

Add a private network.

Which IP Address should we use?

Pick one that's in the block of Private IPv4 Addresses that's been reserved by the Internet Assigned Numbers Authority (IANA).

192.168.1.0/24 will work.

But which IP Address?

When the private network gets created it will need a router, by convention the router will get the first IP address of the 192.168.1.0/24 block, that's 192.168.1.1, so let's choose the next one 192.168.1.2

```
vagrant ssh -c 'ifconfig eth1' | grep -B 1 'inet addr'
eth1      Link encap:Ethernet  HWaddr 08:00:27:7a:84:5e
          inet addr:192.168.1.2  Bcast:192.168.1.255  Mask:255.255.255.0

ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=0.420 ms
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=0.413 ms
^C
--- 192.168.1.2 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.413/0.416/0.420/0.003 ms
```

Why does that work?

Let's look at the routes available to the host again but look for the 192.168.1.0/24 block.

```
netstat -rn | grep 192.168.1
192.168.1          link#22            UC              2        0 vboxnet
192.168.1.2        8:0:27:7a:84:5e    UHLWI           0        2 vboxnet   1145
```

Looks like virtualbox has attached some kind of gateway named link#22, so any packets heading for the IP addresses in the 192.168.1.0/24 block will be routed through the link#22 gateway.

## Hack session

* Generate an SSH key
* Get the key into the guest machine
* Get the key into the authorized keys file

```
ssh-keygen -q -b 2048 -t rsa -N '' -f id_rsa
ssh-keyscan 192.168.1.2 >> ~/.ssh/known_hosts
vagrant_private_key='.vagrant/machines/default/virtualbox/private_key'
scp -i $vagrant_private_key id_rsa.pub vagrant@192.168.1.2:~
ssh -i $vagrant_private_key vagrant@192.168.1.2 echo hello world
ssh -i $vagrant_private_key vagrant@192.168.1.2 ls
ssh -i $vagrant_private_key vagrant@192.168.1.2 cat id_rsa.pub
ssh -i $vagrant_private_key vagrant@192.168.1.2 cat .ssh/authorized_keys
ssh -i $vagrant_private_key vagrant@192.168.1.2 'cat id_rsa.pub >> .ssh/authorized_keys'

ssh -i id_rsa vagrant@192.168.1.2
vagrant@vagrant:~$ ls
id_rsa.pub
vagrant@vagrant:~$ rm id_rsa.pub
```