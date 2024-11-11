**Proctor Summary**

- Explore the network
- Make a manual change
- See how hard it is


# Managing Networks the Hard Way

Let's explore the initial state. It's much like many traditional network setups: some devices, some monitoring and not a lot of documentation.

## Network

Let's take a look at our devices:

```
pushd network
clab inspect
INFO[0000] Parsing & checking topology file: autocon2.clab.yml 
+---+----------------------+--------------+-----------------------------------+---------------+---------+---------------+--------------+
| # |         Name         | Container ID |               Image               |     Kind      |  State  | IPv4 Address  | IPv6 Address |
+---+----------------------+--------------+-----------------------------------+---------------+---------+---------------+--------------+
| 3 | clab-autocon2-srl1   | fb9cd192b40e | ghcr.io/nokia/srlinux:24.7.2      | nokia_srlinux | running | 172.24.0.7/24 | N/A          |
| 4 | clab-autocon2-srl2   | 1a5c96567133 | ghcr.io/nokia/srlinux:24.7.2      | nokia_srlinux | running | 172.24.0.8/24 | N/A          |
+---+----------------------+--------------+-----------------------------------+---------------+---------+---------------+--------------+
```

You can see that we have two Nokia SRLinux devices running in the network. Let's inspect one of them by ssh'ing into `clab-autocon2-srl1`.

> [!TIP]
> 
> **username** admin
> **password** NokiaSrl1!  

```
ssh admin@clab-autocon2-srl1
Warning: Permanently added 'clab-autocon2-srl1' (ED25519) to the list of known hosts.
................................................................
:                  Welcome to Nokia SR Linux!                  :
:              Open Network OS for the NetOps era.             :
:                                                              :
:    This is a freely distributed official container image.    :
:                      Use it - Share it                       :
:                                                              :
: Get started: https://learn.srlinux.dev                       :
: Container:   https://go.srlinux.dev/container-image          :
: Docs:        https://doc.srlinux.dev/24-7                    :
: Rel. notes:  https://doc.srlinux.dev/rn24-7-2                :
: YANG:        https://yang.srlinux.dev/v24.7.2                :
: Discord:     https://go.srlinux.dev/discord                  :
: Contact:     https://go.srlinux.dev/contact-sales            :
................................................................

(admin@clab-autocon2-srl1) Password:
Using configuration file(s): ['/home/admin/.srlinuxrc']
Welcome to the srlinux CLI.
Type 'help' (and press <ENTER>) if you need any help using this.

--{ running }--[  ]--
```

Let's inspect the interfaces:

```
A:clab-autocon2-srl1# show interface
=======================================================================================================================================================================
ethernet-1/1 is up, speed 25G, type None
  ethernet-1/1.0 is up
    Network-instances:
      * Name: default (default)
    Encapsulation   : null
    Type            : routed
    IPv4 addr    : 192.168.0.1/30 (static, preferred, primary)
    IPv6 addr    : 2002::c0a8:0/127 (static, preferred, primary)
    IPv6 addr    : fe80::18c7:ff:feff:1/64 (link-layer, preferred)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
mgmt0 is up, speed 1G, type None
  mgmt0.0 is up
    Network-instances:
      * Name: mgmt (ip-vrf)
    Encapsulation   : null
    Type            : None
    IPv4 addr    : 172.18.0.7/16 (dhcp, preferred)
    IPv6 addr    : fe80::42:acff:fe12:7/64 (link-layer, preferred)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
=======================================================================================================================================================================
Summary
  0 loopback interfaces configured
  1 ethernet interfaces are up
  1 management interfaces are up
  2 subinterfaces are up
=======================================================================================================================================================================
```

We can see that this device has two active interfaces: `mgmt0` and `ethernet-1/1`. `mgmt0` is the interface we just ssh'd in on, `ethernet-1/1` is connected to our other device in the `192.168.0.0/32` subnet. We can confirmed the link to `clab-autocon2-srl2` with LLDP:

```
A:clab-autocon2-srl1# show system lldp neighbor
    +--------------+-------------------+----------------------+---------------------+------------------------+----------------------+---------------+
  |     Name     |     Neighbor      | Neighbor System Name | Neighbor Chassis ID | Neighbor First Message | Neighbor Last Update | Neighbor Port |
  +==============+===================+======================+=====================+========================+======================+===============+
  | ethernet-1/1 | 1A:A1:03:FF:00:00 | clab-autocon2-srl2   | 1A:A1:03:FF:00:00   | 42 seconds ago         | 12 seconds ago       | ethernet-1/1  |
  +--------------+-------------------+----------------------+---------------------+------------------------+----------------------+---------------+
```

Let's ping it across the `ethernet-1/1` interface to confirm connectivity.

```
A:clab-autocon2-srl1# ping 192.168.0.2 network-instance default
Using network instance default
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=67.9 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=4.29 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=3.95 ms
```

Great, the simple network is up and running. Let's have a look at our monitoring.

## Updating the network, the hard way!

Organizations are turning to network automation for many reasons including being able to change the network faster, reducing manual errors, compliance and more. The majority of the industry is just getting started though and for many teams changing the network still means the same old process:

1. Receieve a ticket in the ITSM system
2. Figure out what changes are needed to satisfy the ticket
3. (Sometimes) submit the changes for approval
4. SSH into the devices and make the changes manually
5. Pray!

Let's try one out in our network. Our imaginary company is extremely constrained on IP address space and that /30 between the two devices is just too big! We've been asked to claw back a single IP address by moving to a /31. Let's roll up our sleeves.

> [!TIP]
> 
> If you'd rather skip the manual steps, this command will prepare your network for the next section:  
> `./3_start_network.sh 2_introducing_intent_based`

First on `clab-autocon2-srl1`

```
--{ running }--[  ]--
A:clab-autocon2-srl1# enter candidate

--{ candidate shared default }--[  ]--
A:clab-autocon2-srl1# delete /interface ethernet-1/1 subinterface 0 ipv4 address 192.168.0.1/30

--{ * candidate shared default }--[  ]--
A:clab-autocon2-srl1# set / interface ethernet-1/1 subinterface 0 ipv4 address 192.168.0.0/31

--{ * candidate shared default }--[  ]--
A:clab-autocon2-srl1# commit now
 
All changes have been committed. Leaving candidate mode.
```

Use `Ctrl+D` to exit `clab-autocon2-srl1`. Now on `clab-autocon2-srl2`

```
--{ running }--[  ]--
A:clab-autocon2-srl2# enter candidate

--{ candidate shared default }--[  ]--
A:clab-autocon2-srl2# delete / interface ethernet-1/1 subinterface 0 ipv4 address 192.168.0.2/30

--{ * candidate shared default }--[  ]--
A:clab-autocon2-srl2# set / interface ethernet-1/1 subinterface 0 ipv4 address 192.168.0.1/31

--{ * candidate shared default }--[  ]--
A:clab-autocon2-srl2# commit now
```

And now let's test connectivity. On `clab-autocon2-srl2`:

```
--{ + running }--[  ]--
A:clab-autocon2-srl2# ping -c 4 192.168.0.1 network-instance default
Using network instance default
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=68.2 ms
64 bytes from 192.168.0.1: icmp_seq=2 ttl=64 time=3.92 ms
64 bytes from 192.168.0.1: icmp_seq=3 ttl=64 time=2.83 ms
```

Use `Ctrl+D` to exit `clab-autocon2-srl2`

Phew! 8 commands to apply the changes and 1 command to confirm them. Unfortunately that's not all:

1. If this were a real network, you might have needed to let the monitoring team know there were some changes coming, or accept some alerts going off.
2. We also need to go back and update the network documentation now, if it's exists. Otherwise how future engineers will know what they are getting themselves into when they SSH into these devices.
3. We did a quick ping to check connectivity, but how can we confirm that our devices are correctly (and securely) configured?
4. If we're ever audited, we may be asked to show the reason why this change was made and by whom. How?

Even with this trivial network change that's a lot to worry about, with plenty of surface area for us to fat finger a command or forget an important step. If only there were a better way!