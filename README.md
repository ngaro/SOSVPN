# SOSVPN
Create a VPN that tunnels over `socat` which tunnels over `ssh` to evade extremely strict firewalls

# __THIS BRANCH CONTAINS SCRIPTS THAT DON'T WORK (YET) ! DO NOT TRY TO USE THEM !__

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

## Example assumptions about your network setup
(Change the settings to match your usecase)
### Server
- Has a user named `someuser`
- Has the ip `1.2.3.4`
- Has a networkcard named `eths` connected to the internet
- Has a sshserver running on `tcp/22789` which is reachable from the internet.<br>(It doesn't matter if everything else is blocked)
- Is not yet using the virtual networkdevice `tun0` for other purposes
- Is not yet connected to a network that overlaps the `192.168.255.0/24` range
- Is not yet listening on `tcp/22001`
- Uses the DNS servers `5.6.7.8` and `9.10.11.12`
### Client
- Has a networkcard named `ethc` connected to the internet
- Can create a ssh-connection with the server.<br>(It doesn't matter if everything else is blocked)
- Is not yet using the virtual networkdevice `tun0` for other purposes
- Is not yet connected to a network that overlaps the `192.168.255.0/24` range
- Is not yet listening on `tcp/22002`
- Uses the gateway `13.14.0.254`
- Is on a network containing subnets `13.14.0.0/16` and `15.16.17.0/24` that has the name `mylocal.net`
- Is using the DNS servers `15.16.17.1` and `15.16.17.2`

## Creating the VPN on the client
- Run `screen` because we'll need to run different things at the same time
- Run `ssh -L 22002:127.0.0.1:22001 -p 22789 someuser@1.2.3.4` inside this screen session<br>A `ssh` session will be opened in window `0` of screen:
  - We can now use this session to run commands on the server without having physical access.
  - The `-L` option is used for local portwarding to make sure that when the server sends traffic to itself on `TCP/22001` it will be encrypted inside ssh-packets and end up at `TCP/22002` on the client.
- Run `screen` inside this ssh-session.<br>We now have a screen session on the client with 1 window (number `0`) and inside it a screen session on the server.<br>This also has 1 window that is numbered `0`.<br>This screen session is also needed because we run different things at the same time in the server.
- Run `sudo socat -d -d TCP-LISTEN:22001,reuseaddr TUN:192.168.255.1/24,up` inside window `0` of the server on screen. The result:
  - A virtual network device `tun0` with ip `192.168.255.1` and netmask `255.255.255.0` will be created
  - The system will start listening to `TCP/22001` for traffic from the server itself and interpret it as traffic from a virtual network with the descriptions above.
  - This traffic will be sent to the system over `tun0`
  - The portforwarding means that it's actually listening to traffic sent to `TCP/22002` on the client by the client itself
  - It works bi-directional, so traffic sent out by the system to a system on `192.168.255.0/24` will leave using `tun0`, will be packed into TCP-packets and these will be tunneled over the ssh-connection to finally end up at `TCP/22002` on the client.
(`-d -d` is used for reasonably verbose debug info in case something goes wrong)
- Press __ctrl-a__ followed by a __a__ and a __c__ to create another window (number `1`) in the screen session on the server.<br>We will use this to make sure that traffic from the client meant for the outside world will be passed on by the system and is NAT-ed:
```
sudo iptables -t nat -A POSTROUTING -s 192.168.255.0/24 -j MASQUERADE
sudo iptables -A FORWARD -i tun0 -o eths -j ACCEPT
sudo iptables -A FORWARD -o tun0 -i eths -j ACCEPT
```
- Press __ctrl-a__ followed by a __c__ to create another window (number `1`) in the screen session on the client.<br>Run `sudo socat -d -d  TCP:127.0.0.1:22002 TUN:192.168.255.2/24,up` here. This will create the other endpoint of the tunnel.<br>Note that the ip is different (`192.168.255.2`) but still in the same network as the one on the server (`192.168.255.0/24`).<br>This is necessary to make sure that communication is possible.
- Press __ctrl-a__ followed by a __c__ to create another window (number `1`) in the screen session on the client.<br>We will use this make sure traffic is routed correctly:
```
# Make sure that the server itself is not routed over the VPN:
sudo route add 1.2.3.4 gw 13.14.0.254
# Make sure that the traffic sent to the gateway is sent out using the real network device:
sudo route add 13.14.0.254 dev ethc
# Make sure that traffic meant for our network stays in our network:
for net in 13.14.0.0/16 15.16.17.0/24 ; do sudo route add -net $net gw 13.14.0.254 ; done
# Make sure that all other traffic is sent over the VPN instead of the regular network
sudo route add default gw 192.168.255.1 && sudo route del default gw 13.14.0.254
```
### Setup DNS for the VPN
You probably want your client to resolve internal names using the dnsservers on your network, and the rest of the network using external dnsservers (the ones on your server).<br>You can do this like this:
- Install bind (`sudo apt install bind9` on debian-based systems)
- Add the following to `/etc/bind/named.conf.local`:
```
zone "mylocal.net" IN {
  type forward;
  forwarders {
    15.16.17.1; 15.16.17.2;
  };
};
```
- Place the following in the `options {};` section in `/etc/bind/named.conf.options`:
```
forwarders { 5.6.7.8; 9.10.11.12; };
allow-recursion { 127.0.0.1; };
querylog yes;
```
- Restart your local dns-server: `systemctl restart named`
- Make sure you use `127.0.0.1` as dns-server. (The "clean" procedure differs from system to system, the "ugly" sollution of commenting everything in `/etc/resolv.conf` and adding the line `nameserver 127.0.0.1` should work everywhere)
## Destroying the VPN
The short version: Use all the steps you used to create the VPN in reverse.<br><br>The long version:
- Attach the screen session with  `screen -r` if it's no longer on your terminal, jump to window `2` with __ctrl-a__ followed by __2__.<br>You can reset the routing on the client here:
```
sudo route del default gw 192.168.255.1 && sudo route add default gw 13.14.0.254
for net in 13.14.0.0/16 15.16.17.0/24 ; do sudo route del -net $net gw 13.14.0.254 ; done
sudo route del 13.14.0.254 dev ethc
sudo route del 1.2.3.4 gw 13.14.0.254
exit
```
- Jump to window `0` with __ctrl-a__ followed by __0__ where the screen session in the ssh-session is running.<br>Now jump now to window `1` in this screen session with __ctrl-a__ followed by __a__ followed by __1__ and reset `iptables` here:
```
sudo iptables -t nat -D POSTROUTING -s 192.168.255.0/24 -j MASQUERADE
sudo iptables -D FORWARD -i tun0 -o eths -j ACCEPT
sudo iptables -D FORWARD -o tun0 -i eths -j ACCEPT
exit
```
- You will now end up in window `0` of the screen-session on the server, press __ctrl-c__ here to kill socat. This will also kill the endpoint of the tunnel at the client. Now run the `exit` command a couple of times until all windows in both screen sessions are killed.
- Use your original DNS configuration again by:
  -  Stopping bind with `systemctl stop named`
  -  Changing your dnsservers back to `15.16.17.1` and `15.16.17.2`.<br>Again, multiple ways to do this are possible. Placing `nameserver 15.16.17.1` and `nameserver 15.16.17.2` in `/etc/resolv.conf` and commenting out the rest should work as "ugly" solution.

<br><br>SOSVPN is created by Nikolas Garofil and licensed under GPLv3, see the `License` file for details.
