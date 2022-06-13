# Provisioning RKE clusters inside vSphere infrastructure using Rancher
*Requirements: Rancher >= v2.3.6*

## This is a guide showing how to configure Rancher to provision RKE clusters automatically using vSphere 

### Create a new template virtual machine on vSphere using the following configurations:
1. Install docker and cloud-init 
2. Add docker user on group docker
`usermod -aG docker <user_name>`
3. Disable swap
`swapoff -a
&& sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`
4. Apply the following sysctl settings:
`sysctl -w net.bridge.bridge-nf-call-iptables=1`
5. Create a file disabling coud-init initial network configuration
`vim /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`
  with the following content:
`network: {config: disabled}`
6. Reset machine-id
```
truncate -s 0 /etc/machine-id
rm /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id
```
7. Start and enable cloud-init modules
```
systemctl start cloud-init-local.service && systemctl enable cloud-init-local.service
systemctl start cloud-init.service && systemctl enable cloud-init.service
systemctl start cloud-config.service && systemctl enable cloud-config.service
systemctl start cloud-final.service && systemctl enable cloud-final.service
```
8. Reboot the virtual machine, turn it off and create a template inside vSphere


By the way, this is the way Rancher communicates with vSphere in order to provision clusters (it uses docker machine and cloud-init in order to do this):  
![Arquitetura](https://i.imgur.com/5yYbRvX.png)


### If your environment DOES have DHCP
Then it's all good pretty much. Now you just need to create the Cloud Credential and Node Templates for vSphere inside Rancher

1. Inside Rancher go to `User > Node Templates > vSphere`
2. On the Cloud Credentials click on `Add New` and insert your vSphere Credentials. **Add an user with enough privileges to create and delete infrastructure inside vSphere.**
3. Configure the rest of the parameters on the Node Template matching your environment.

### If your environment DOESN'T have DHCP (hard mode)
We are going to use vSphere Network Protocol Profile, vApp and cloud-init to get things done here:

1. First off we need to create a vSphere Network Protocol Profile. Inside vSphere go to `Datacenter > Configure > Network Protocol Profiles and click Add`.
2. Put a name on it and assign at least one port group
3. Then click on `IPv4` and define the Subnet, Gateway, DNS Server Addresses, IP Pool and IP Pool Range. IP Pool and IP Pool Range are the parameters that will make possible to assign an unique IP address to each one of the created nodes. 
![Network Protocol Profile](https://i.imgur.com/BmyLC7M.png)
4. After you create the Network Protocol Profile you should be able to see that it is assigned to the Port Group that you chose on the tab `Assigned Networks`
5. Go to the template virtual machine and `Enable` vApp on the `Configure` tab. On the `Properties` tab you have to configure the ovf environment variables for IP, Netmask, Gateway and DNS. Make sure that they follow the format: `${param:portgroup}`. You should have something like this:
![vApp](https://i.imgur.com/WI70OJi.png) 
6. Inside Rancher go to `User > Node Templates > vSphere`
7. On the Cloud Credentials click on `Add New` and insert your vSphere Credentials. **Add an user with enough privileges to create and delete infrastructure inside vSphere.**
8. Configure the rest of the parameters on the Node Template matching your environment.
9. In the Cloud Config YAML tab you have to add a script that will use the variables created on the ovf environment by vSphere and configure the OS network. The below example is from a SUSE Linux 15SP2 using Network Service:
```
#cloud-config
write_files: 
  - path: /root/test.sh
  content: | 
    #!/bin/bash 
    vmtoolsd --cmd 'info-get guestinfo.ovfEnv' > /tmp/ovfenv 
    IPAddress=$(sed -n 's/.*Property oe:key="guestinfo.interface.O.ip.O.address" oe:value="\([^"]*\).*/\1/p' /tmp/ovfenv) 
    SubnetMask=$(sed -n 's/.*Property oe:key="guestinfo.interface.O.ip.0.netmask" oe:value="\([^"]*\).*/\1/p' /tmp/ovfenv) 
    Gateway=$(sed -n 's/.*Property oe: key="guestinfo.interface.0.route.0.gateway" oe:value="\([^"]*\).*/\1/p' /tmp/ovfenv) 
    DNS=$(sed -n 's/.*Property oe:key="guestinfo.dns.servers" oe:value="\([^"]*\).*/\1/p' /tmp/ovfenv) 
 
    echo "ONBOOT='yes'
    IPADDR='$IPAddress/$SubnetMask'
    BOOTPROTO='static'
    STARTMODE='auto'" > /etc/sysconfig/network/ifcfg-eth0
    echo "default $Gateway - -" > /etc/sysconfig/network/routes
    
    echo "nameserver $DNS" >> /var/run/netconfig/resolv.conf
    EOF
runcmd: 
 - sudo chmod +x /root/test.sh 
 - sudo bash /root/test.sh 
 - sudo systemctl restart network.service
```
10. Now you have to configure the vApp properties, so that cloud-init can map the network values (IP, mask, DNS and Gateway) from the vApp variables we configured on step 5. You shall have something like this:
![Passing Variables](https://i.imgur.com/9XI5v6C.png)

🎉🎉🎉

Huge thanks to @David-VTUK (https://github.com/David-VTUK) for all the help!

