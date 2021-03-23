# SOSVPN
Create a VPN that tunnels over `socat` which tunnels over `ssh` to evade extremely strict firewalls

__SOSVPN__ ( or SOcatSshVPN ) is a VPN-service that has 2 features that most others lack:
- Only extremely strict firewalls can block it. The only allowed traffic needed is:
  - Outgoing ssh-traffic on the clientside
  - Incoming ssh-traffic on the serverside
- Even a deep packet inspection firewall can't recognize the traffic as VPN-traffic, it will only see ssh-packets

And a couple of features (some, but not all) other VPN-software has:
- Traffic from the client meant for the internal network will stay in the internal network
- All other traffic will be sent over the VPN to the internet with the server acting as gateway

It works by:
- Creating a virtual network
- ... that's tunnelled over a `socat`-tunnel
- ... that's tunneled over a `ssh`-tunnel
This explains the name __SOcatSshVPN__.<br>
Not only is it just as fast as regular ssh-traffic, but by using ssh-compression you can even make it faster then many other VPN's.

_This is still version 0.1, here I'm only explaining how to set it up manually.
Interpreted (or compiled) code will follow later._

## Example assumptions about your network setup
(Change the settings to match your usecase)
### Server
- Has the ip `1.2.3.4`
- Has a networkcard named `eths` connected to the internet
- Has a sshserver running on `tcp/22` (this is the default ssh-port) which is reachable from the internet.<br>(It doesn't matter if everything else is blocked)
- Is not yet using the virtual networkdevice `tun0` for other purposes
- Is not yet connected to a network that overlaps the `192.168.255.0/24` range
- Is not yet listening on `tcp/22001`
- Uses the DNS servers `5.6.7.8` and `9.10.11.12`
### Client
- Has the ip `13.14.15.16`
- Has a networkcard named `ethc` connected to the internet
- Can create a ssh-connection with the server.<br>(It doesn't matter if everything else is blocked)
- Is not yet using the virtual networkdevice `tun0` for other purposes
- Is not yet connected to a network that overlaps the `192.168.255.0/24` range
- Is not yet listening on `tcp/22002`
- Uses the gateway `13.14.0.254`
- Is on a network containing subnets `13.14.0.0/16` and `15.16.17.0/24`
- Is using the DNS servers `15.16.17.1` and `15.16.17.2`

## Preliminary setup
- Install the necessary software (`iptables`, `socat`, `ssh` and `screen`) on the server.<br>On debian-based system this can be done with: `apt install iptables socat ssh screen`
- Make sure the ssh server runs on the server:<br>Check `systemctl status ssh` and run `systemctl restart ssh` if you didn't see `active (running)` in the output.
<br><br>_From now on you never need physical access to the server anymore._<br><br>
- Install the necessary software (`route`, `socat`, `ssh` and `screen`) on the client.<br>On debian-based system this can be done with: `apt install net-tools socat ssh screen`
## TODO: Describe how to create/start the VPN
## TODO: Describe how to stop the VPN
## TODO: Describe how to setup DNS
