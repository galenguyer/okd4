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

under firewall>nat, set up some port forwarding rules for ports like 22, 80, and 443 to 192.168.1.200. this way you can reach your services vm.

## okd4-services
### installation
grab the latest centos 8 stream installation iso from [the centos downloads page](https://www.centos.org/download/). 

create a new vm named okd4-services. use the centos iso. ensure it has at least 100gb of storage space (this number may grow). also give it 4 cores and 4gb of ram. also ensure it's on your internal network, not your external one. later we'll set up nat to this vm through pfsense.

from the pfsense web ui, go to services>dhcp server and add a static mapping for the okd4-services vm interface. give it the ip 192.168.1.200 and the hostname okd4-services.

install centos 8. 
  under networking, enable the interface and set the hostname to okd4-services.
    you might have to add another dns server actually. i like 1.1.1.1
  for software selection, you can do server or server with gui. i don't like guis so i go without. also install the guest agent because yknow this is a vm
  for the partitioning, use custom partitioning. use the defaults, and then remove the home partition. set the root partition to fill the whole disk. yay space

add epel-release and git and upgrade (won't be much cause it's stream)
```
dnf install -y epel-release git
dnf update -y
```

## bootstrap and masters and workers, oh my
grab the latest coreos iso from [the coreos download page](https://getfedora.org/coreos/download/).

all these machines will need 4 cores, 16gb of ram, and 120gb of hard drive space. monsters, i know

i'm doing okd-bootstrap, okd-master-{1..3}, and okd-worker-{1..2}. you can drop or add workers as you want

| name | ip |
| --- | --- |
| okd4-master-01 | 192.168.1.201 |
| okd4-master-02 | 192.168.1.202 |
| okd4-master-03 | 192.168.1.203 |
| okd4-worker-01 | 192.168.1.204 |
| okd4-worker-02 | 192.168.1.205 |
| okd4-bootstrap | 192.168.1.210 |

assign all these ip addresses in the pfsense web ui

## ok, back to services

### bind (dns)
install bind:
```
dnf install -y bind bind-utils
```
update the files in [okd4-services/etc/named.conf](okd4-services/etc/named.conf) and [okd4-services/named/](okd4-services/named/) as needed, if you changed the node ip addresses in pfsense. you'll probably want to change the base domain as well. the `okd4` just before the base domain is the cluster name, call it whatever you like. double check this because dns will fuck you.

once you've double and triple checked your named configs, copy them into /etc/ on services. i've nicely set up the directory structure for you, literally just copy them over how they are and it should work. then start the dns service
```
systemctl enable --now named
systemctl status named
```

create firewall rules:
```
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --reload
```

update your interface to use localhost as your primary dns server with `nmtui` and then restart networking: `systemctl restart NetworkManager`

### haproxy
install haproxy:
```
dnf install -y haproxy
```

make sure [okd4-services/etc/haproxy/haproxy.cfg](okd4-services/etc/haproxy/haproxy.cfg) looks good then copy it to /etc/haproxy/haproxy.cfg

enable and start haproxy:
```
setsebool -P haproxy_connect_any 1 # for SELinux
systemctl enable --now haproxy
systemctl status haproxy
```

add firewall rules:
```
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=22623/tcp
firewall-cmd --permanent --add-service=80/tcp
firewall-cmd --permanent --add-service=443/tcp
firewall-cmd --reload
```

### apache/httpd
install httpd:
```
dnf install -y httpd
```

change the listen port to 8080 in `/etc/httpd/conf/httpd.conf`

enable, start, and add firewall rules for httpd
```
setsebool -P httpd_read_user_content 1
systemctl enable --now httpd
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

## oc client and installer
~~(still on services) grab the latest client and installer from the [okd release page](https://github.com/openshift/okd/releases).~~

**DO NOT grab the latest version. it's all fucky wucky. Use [CoreOS 32](https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/32.20201104.3.0/x86_64/fedora-coreos-32.20201104.3.0-live.x86_64.iso) instead.**

extract them with tar

move the binaries into /usr/local/bin
```
mv kubectl oc openshift-install /usr/local/bin/
```
verifiy installation and versions:
```
oc version
openshift-install version
```

if openshift-install gives you the error `openshift-install: error while loading shared libraries: libvirt-lxc.so.0: cannot open shared object file: No such file or directory`, install libvirt:
```
dnf install -y libvirt
```

### set up the installer
generate an ssh key with `ssh-keygen`

some notes about `installation/install-config.yaml`:
- documentation is available on [https://docs.okd.io/](https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html#installation-bare-metal-config-yaml_installing-bare-metal)
- you'll have to update the ssh key
- you don't need to change the pull secret but you can
- it's really self explanatory, i don't know what else to put here

generate manifests from the install-config:
```
openshift-install create manifests --dir=./installation
```

by default masters are schedulable. you can edit `installation/manifests/cluster-scheduler-02-config.yml` and set mastersSchedulable to false. if you do this, you'll have to edit haproxy.cfg to not route to masters as well.

next create inition configs:
```
openshift-install create ignition-configs --dir=./installation
```

### host installation configs on httpd

make the folder for the files with `mkdir /var/www/html/okd4`

copy the installation folder contents to that folder and make them readable:
```
sudo cp -Rv installation/* /var/www/html/okd4/
sudo chown -Rv apache: /var/www/html/
sudo chmod -Rv 755 /var/www/html/
```

test the hosting with `curl localhost:8080/okd4/metadata.json`

download the [fedora coreos bios images](https://getfedora.org/coreos/download/) and shorten the file names:

**NOTE:: Don't use the latest ones. Use the ones specified below. Latest WILL fail as of writing thing.**
```
curl https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/32.20201104.3.0/x86_64/fedora-coreos-32.20201104.3.0-metal.x86_64.raw.xz -o /var/www/html/okd4/fcos.raw.xz
curl https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/32.20201104.3.0/x86_64/fedora-coreos-32.20201104.3.0-metal.x86_64.raw.xz.sig -o /var/www/html/okd4/fcos.raw.xz.sig
chown -Rv apache: /var/www/html/
chmod -Rv 755 /var/www/html/
```

## node ignition
### bootstrap node
start the okd4-bootstrap node and hit tab to edit the boot config. 

add the following options:
```
coreos.inst.install_dev=/dev/sda coreos.inst.image_url=http://192.168.1.200:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.1.200:8080/okd4/bootstrap.ign
```

### master nodes
start the master nodes and hit tab to exit their boot configs

add the following options:
```
coreos.inst.install_dev=/dev/sda coreos.inst.image_url=http://192.168.1.200:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.1.200:8080/okd4/master.ign
```

### worker nodes
start the worker nodes and hit tab to exit their boot configs

add the following options:
```
coreos.inst.install_dev=/dev/sda coreos.inst.image_url=http://192.168.1.200:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.1.200:8080/okd4/worker.ign
```

the worker nodes getting an internal server error from api-int.okd4.stonewall.lan is expected, don't panic

you can monitor the bootstrap process with
```
openshift-install --dir=./installation wait-for bootstrap-complete --log-level=info
```
