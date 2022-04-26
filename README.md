# wireguard_playbook
Ansible Playbook for WireGuard VPN

This is simple Ansible Playbook to automatically provision WireGuard on all machines defined in inventory.
The emphasis for this playbook is to define all required parameters in YAML file and let ansible to generate private/public keys and then configure every peer accordinly, because otherwise it is quite burdensome activity to do so manually.

### Simple overview of peers ###
```
   VPS in Datacenter
   ┌────────────────┐
   │WAN IP: 1 1.1.1 │◄─────┐
   │WG IP: 10.0.0.1 │      │
   └────────────────┘      │           Desktop at home
                           │         ┌─────────────────┐
                           ├─────────► WAN IP: 1.1.1.2 ├──────────┐
                           │         │ WG IP: 10.0.0.2 │          │
                           │         └─────────────────┘          │
                           │                                      │
  Laptop on the road       │                          LAN at home │
 ┌──────────────────────┐  │                          ┌───────────▼──────────┐
 │WAN IP: 192.168.1.101 │  │                          │CIDR: 192.168.123.0/24│
 │WG IP: 10.0.0.3       ├──┘                          │                      │
 └──────────────────────┘                             └──────────────────────┘

```

To configure such simple WireGuard setup it's enough to define following setup in site.yml
```yaml
 ---
 - hosts: all
   gather_facts: true
   become: yes
   vars:
     wireguard_iface: "wg0"
     # This playbook saves generated private keys to disk, toggle this if you wish to generate new keys when replaying this playbook
     # otherwise it will use already generated keys.
     rotate_keys: false
     # Toggle this if you wish to write Private IP in as wireguard peer endpoint (Useful for testing with couple of VMs)
     peer_private_ip: false
     wireguard_peers:
     # IMPORTANT: here server, desktop, laptop are actual hostnames of your machines! define them correctly, otherwise playbook will fail to find IP addresses
     # to use as actual WireGuard endpoints!
       server:
     # list of IP addresses to assign on wg0 interface, ip4/6 private IPs.
         ips:
           - 10.1.0.1/32
           - fd00:1337::1/128
     # can leave this out and get a random port assigned - CAUTION: opening FW is out of scope for this playbook!
         port: 41294
       desktop:
         ips:
           - 10.1.0.2/32
           - fd00:1337::2/128
         port: 25123
      # This optional IP range will be added as route to server/laptop peers to route to this LAN behind desktop machine.
      # CAUTION - configuring desktop firewall to do IP routing is out of scope for this playbook!
         allowed_ips:
           - 192.168.123.0/24
       laptop:
         ips:
           - 10.1.0.3/32
           - fd00:1337::3/128
         port: 15123
      # Since laptop is on mobile network without own WAN IP, laptop will connect to other WG peers to establish VPN connection,
      # but that usually comes with issues when other peers want to connect to laptop which is behind NAT and might not be reachable.
      # So PersistentKeepAlive tells laptop to send empty packet to peers notifying them laptop is still online, which helps connection persistency.
         extra_opts:
           PersistentKeepalive: 21
   roles:
     - common
     - wg_install
     - wg_configure
```
As you might have noticed that WAN IP addresses are nowhere defined in this playbook, that's intentional because ansible gathers facts on each machine and uses default route's source IP address as WireGuard peer IP! In case of laptop machine, it has private IP - so it won't be listed as Endpoint on other peers (VPS,desktop), because they cannot physically connect to it. Laptop however will connect to other peers and whole network will be functional.

Note: This setup requires Python netaddr module to be installed on ansible control host, because it evaluates if certain peers default IP is public (VPS,Desktop) or private, like on laptop connected via mobile network.
Install required dependency:
```
apt-get install python3-netaddr
```
Edit the example site definition to suite your needs and run the ansible playbook
```
ansible-playbook -i inventory site.yml
```
And the best part - adding new machines to this setup is just a matter of adding them to yaml definition and re-running playbook - each peer will be configured with needed keys and endpoint IPs.
