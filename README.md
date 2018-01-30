## Synopsis

This is a very simple reference topology for Cisco VIRL/CML demonstrating how to leverage 'Server' linux nodes in your simulation.

## Topology

We will be using three 'server' linux nodes connected by unmanaged L2 switch
![Alt text](server-nodes/img/server_topo.png?raw=true "VIRL/CML Server Node Topology")

## Motivation

'Server' nodes are a little tricky when it comes to creating topologies with them.
When the nodes are first created, they come with no default configuration and when you start your simulation you will discover that you can not login into them (error claiming invalid credentials). Server nodes are based on LXC technology and are really headless - they do not have physical
discs. When your simulation is spun down, server configuration is lost. 
The way to apply configuraiton to your 'server' nodes is to leverage cloud-config technology, which leverages a simple textual configuration 'recipe' which gets injected into the server node at simulation startup.

By default, eth0 of every node gets assigned to 'management network' that is not routable outside of your VIRL/CML simulation.

There are two ways that you can generate the cloud-init for your newly created nodes. 
1. Use ANK (Automatic NetKit). Head over to Configuration > Generate Configuration. This will auto-generate configs for *every* node in your topology. Beware that this approach has a tendency to overwrite configuration in every single device, so you are risking of loosing your configs.
Especially this is true when you decide that you want to add a 'server' node to your existing topology at a later stage. I always disable the
automatic generation of configurations for nodes other than 'servers':

![Alt text](server-nodes/img/ank_tickbox.png?raw=true "ANK auto-config generate option")

2. Use a simple cloud-config template that you can copy-paste to your notepad, modify placeholders as needed and paste into 'server' node config.
Update <<>> placeholders in the config before applying:

```
#cloud-config
bootcmd:
- ln -s -t /etc/rc.d /etc/rc.local
hostname: <<SERVER_NAME>>
manage_etc_hosts: true
runcmd:
- start ttyS0
- systemctl start getty@ttyS0.service
- systemctl start rc-local
- sed -i '/^\s*PasswordAuthentication\s\+no/d' /etc/ssh/sshd_config
- echo "UseDNS no" >> /etc/ssh/sshd_config
- service ssh restart
- service sshd restart
users:
- default
- gecos: User configured by VIRL Configuration Engine 0.23.10
  lock-passwd: false
  name: cisco
  plain-text-passwd: cisco
  shell: /bin/bash
  ssh-authorized-keys:
  - VIRL-USER-SSH-PUBLIC-KEY
  sudo: ALL=(ALL) ALL
write_files:
- path: /etc/init/ttyS0.conf
  owner: root:root
  content: |
    # ttyS0 - getty
    # This service maintains a getty on ttyS0 from the point the system is
    # started until it is shut down again.
    start on stopped rc or RUNLEVEL=[12345]
    stop on runlevel [!12345]
    respawn
    exec /sbin/getty -L 115200 ttyS0 vt102
  permissions: '0644'
- path: /etc/systemd/system/dhclient@.service
  content: |
    [Unit]
    Description=Run dhclient on %i interface
    After=network.target
    [Service]
    Type=oneshot
    ExecStart=/sbin/dhclient %i -pf /var/run/dhclient.%i.pid -lf /var/lib/dhclient/dhclient.%i.lease
    RemainAfterExit=yes
  owner: root:root
  permissions: '0644'
- path: /etc/rc.local
  owner: root:root
  permissions: '0755'
  content: |-
    #!/bin/sh
    sudo ifconfig eth1 up <<SERVER_IP>> netmask <<XXX.XXX.XXX>>
    sudo route add -net <<NETWORK/MASK>> gw <<GW_IP>> dev eth1
    exit 0
 ```
