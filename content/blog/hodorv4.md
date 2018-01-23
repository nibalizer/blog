+++
title = "Hodor v4"
description = "The fourth iteration of the get-me-a-vm script"
tags = ['scripting', 'bash', 'vm', 'softlayer', 'work']
categories = []
date = "Mon Jan 22 23:32:47 CST 2018"
+++



Several times in the past I've created a script called 'hodor'. This week  created the fourth version of the script. 

Hodor is a script to emit a test vm for me to do something with. Expected usage:


```
$ hodor herpderp
Hodor! Making vm
VM ready! ssh root@herpderp.hodor.nibz.science
```

In past iterations, hodor was written against HP cloud in bash. Then written against HP cloud, IBM cloud, and vexxhost using the shade library in python. Then written again against vexxhost but using bash.

Version four of hodor is written in bash against the IBM Cloud Softlayer vsi infrastructure. It makes heavy use of the bluemix (bx for short) utility. It has dns integration and ssh key integration.

To get started I bought the throwaway domain name ``nibz.science`` and registered it via gandi. Gandi, by the way, is a great registrar and recently updated their user experience to be much better. After everything went through, I set the NS records to point at SoftLayer.


```
$: dig -t ns nibz.science

; <<>> DiG 9.10.3-P4-Ubuntu <<>> -t ns nibz.science
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57026
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nibz.science.              IN     NS

;; ANSWER SECTION:
nibz.science.               IN     NS     ns2.softlayer.com.
nibz.science.               IN     NS     ns1.softlayer.com.

;; Query time: 2 msec
;; SERVER: 127.0.1.1#53(127.0.1.1)
;; WHEN: Fri Jan 19 23:45:33 CST 2018
;; MSG SIZE  rcvd: 90
```

Once that was done, I setup the domain in SL. CLI dns management in IBM Cloud/Softlayer is clunky but effective. 


```
$: bx sl dns zone-create nibz.science
OK
Zone nibz.science was created.
$:
```

From there (after a little time) adding records was easy.


```
$: bx sl dns record-add nibz.science throwaway A 127.0.0.1 --ttl 3600
OK
Created resource record under zone nibz.science: ID=106702065, type=A, record=throwaway, data=127.0.0.1, ttl=3600.
```

With DNS particulars setup, I just had to write a script to boot a vm.

The ``bx sl vs`` command is how you do this. (That expands out to 'bluemix softlayer virtual-server' if anyone was curious.) This is another clunky utility, but I don't blame it too much, there are just a ton of dimensions to virtual server creation and that all comes together in the ``create`` command. One of the reasons for me creating hodor is that I don't want to know or think about ram, datacenter, ubuntu version, etc. I just want a latest ubuntu with enough resources to be able to screw around and I want it now.


```
$ bx sl vs create -H $name -D hodor.nibz.science -c 8 -m 16384 -d dal09 -o UBUNTU_LATEST --disk 100 --key $my_key_id --force --wait 3600

```

Unpacking this a bit. The name and domain here are more or less what you'd expect. 8 vcpu cores and 16 gigs of ram is big enough that if I need more than that I probably can take the time to create a server more manually. ``dal09`` is one of softlayer's datacenters. The ``--key`` option refers to the id of an ssh key added with the ``bx sl security`` command.  ``--force`` creates the server without asking you to confirm. ``--wait 3600`` turns the create operation into a blocking command with a timeout of an hour. In testing, vms took between 3 and 4 minutes to come up.

This command returns some info about the server, including it's id. I grab that id and run the details command to get the ip.


```
$: bx sl vs detail 48197835
Name              Value   
ID                48197835   
guid              0a0babd8-49f3-44d8-b6b4-0c629b7d7eaf   
hostname          gethype-1   
domain            hodor.nibalizer.science   
fqdn              gethype-1.hodor.nibalizer.science   
status            Active   
state             Running   
datacenter        dal09   
os                Ubuntu   
os version        16.04-64 Minimal for VSI   
cpu cores         8   
memory            16384   
public ip         169.54.244.9   
private ip        10.142.169.37   
private network   false   
private cpu       false   
created           2018-01-20T04:27:42Z   
updated           2018-01-20T04:30:53Z   
note                 
vlans             type      number   ID      
                  PUBLIC    846      2248345      
                  PRIVATE   824      2248347      
```

With the ip and id, I'm able to set the dns record. These generally populate through pretty darn quickly, so long as you don't ask too soon and end up caching a negative response.


The script in whole:

```
$: cat hodor.sh 
#!/bin/bash

echo "please have sl set up"
bx sl vs list | grep hodor

name=$1

if [ -z $name ]; then
    echo NEED NAME HODOR
fi

echo "Starting " `date`

id=$(bx sl vs create -H $name -D hodor.nibz.science -c 8 -m 16384 -d dal09 -o UBUNTU_LATEST --disk 100 --key 1063573 --force --wait 3600 | grep ID | awk '{print $NF}')

ip=$(bx sl vs detail 48196973 | grep 'public ip'| awk '{print $NF}')
bluemix sl dns record-add nibz.science $name.hodor A $ip --ttl 3600
echo "Ending " `date`
echo "SL id is" $id
echo "hostname is" $name.hodor.nibz.science
echo "ssh root@${ip}"
```


What isn't reflected in this work so far, is automatic cleanup or caching. The holy grail, in my mind, is Puppet's vmpooler. The pooler keeps a quota of hot, already prepared and functional vms around, so that checking one out is effectively instant. It also has the concept of leases, by default they'll be cleaned up 24 hours after checkout unless the user presses a button or hits an api to extend the lease. 

I'd love to build something like that (vmpooler is too tied to vmware to try to just write another driver) but for me, saving just a few minutes isn't worth the expenditure of having unused vms sitting around just in case I decide I do want them.

However, I could build a cleanup daemon that would delete hodor vms 24 hours after creation. In past use of the script these throwaway test vms tend to pile up and sit around.


```
$: ./hodor.sh jenkins
please have sl set up
Starting  Sat Jan 20 00:04:47 CST 2018
OK
Created resource record under zone nibz.science: ID=106709115, type=A, record=jenkins.hodor, data=169.54.244.2, ttl=3600.
Ending  Sat Jan 20 00:08:34 CST 2018
SL id is 48200667
hostname is jenkins.hodor.nibz.science
ssh root@169.54.244.2
```


References:

SSH Keys in Softlayer: https://knowledgelayer.softlayer.com/procedure/ssh-keys-0
VM Pooler: https://github.com/puppetlabs/vmpooler

NOTE: the --key stuff actually doesn't work for me (follow up with kelley)
