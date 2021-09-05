# okd4
this is going to hurt

anyways, okd4 on proxmox using a single ip address. wish me luck

## vnets
you'll need two vnets for this, one exposed to the internet and one for internal routing. my interfaces file is at [stonewall/etc/network/interfaces](stonewall/etc/network/interfaces). you'll probably have to change some interface names to get stuff to work.

## pfsense
pfsense is needed for routing within the vms and nat. 

### installation
download the latest amd64 dvd iso from [the website](https://www.pfsense.org/download/).

make sure to set the guest os type to other (pfsense is bsd-based). 1 cores and 2gb of ram is fine.

make sure to give it two network interfaces - one on your public network and one on the internal vnet.

run through the installation with the default values, they'll work fine.

### configuration, part the first
once the installation and reboot is complete, hit `8` to hop into a shell. run `pfctl -d` to disable the firewall temporarily. connect to https://[server ip] and complete the setup wizard. the default login credentials are `admin:pfsense`. 

set the hostname to pfsense and the domain to whatever you want, really. i'm going to use `stonewall.lan` in this example. set the primary dns server to `192.168.1.200`.

once you reload, you'll have to run `pfctl -d` to disable the firewall again. you'll have to run this multiple times every time you reload the firewall throughout the next steps.

under firewall>rules, create an allow rule for whatever port you want to change the pfsense admin panel port to run on. i use port 9443 because i don't think i'll ever use that in something else. make sure to apply your changes.

under system>advanced, change the web admin port to whatever port you opened. restart stuff and connect to the web interface on the new port for the next steps.

under services>dhcp server, change the range so you can give static leases from 192.168.1.200-192.168.1.255. also make sure 192.168.1.200 is set as a dns server.

## okd4-services
grab the latest centos 8 stream installation iso from [the centos downloads page](https://www.centos.org/download/). 

create a new vm named okd4-services. use the centos iso. ensure it has at least 100gb of storage space (this number may grow). also give it 4 cores and 4gb of ram. also ensure it's on your internal network, not your external one. later we'll set up nat to this vm through pfsense.

from the pfsense web ui, go to services>dhcp server and add a static mapping for the okd4-services vm interface. give it the ip 192.168.1.200 and the hostname okd4-services.

install centos 8. 
  under networking, enable the interface and set the hostname to okd4-services.
    you might have to add another dns server actually. i like 1.1.1.1
  for software selection, you can do server or server with gui. i don't like guis so i go without. also install the guest agent because yknow this is a vm
  for the partitioning, use custom partitioning. use the defaults, and then remove the home partition. set the root partition to fill the whole disk. yay space

