# Homelab documentation

## 📑 Table of Contents
1. [Requirements](#requirements)
2. [Samba](#-samba)
3. [Unbound](#-unbound)
4. [Keepalived](#-keepalived)
5. [Docker Compose](#-docker-compose)

## Requirements

Docker, samba, unbound and keep alived need to be installed before starting.

`sudo apt install unbound`

`sudo apt install keepalived`


## 📁 Samba

In root of repository copy `smb.conf` file to the `/etc/samba/`. This file containes sample of one share that requires user login and second one is guest folder that everyone can read but nobody can write to it. It is important to note in this simple samba setup every samba user needs to have their unix account because of the permission managment for folders. After creating unix user said user can be added to samba with this command `sudo smbpasswd -a user_name`. Everytime there is a change to config you must restart services with commands `sudo systemctl restart smbd` and `sudo systemctl restart nmbd`. If you want to start samba at boot then you need to do `sudo systemctl enable smbd` and `sudo systemctl enable nmbd`

## 🔎 Unbound
In root of repository copy `pi-hole.conf` file to the `/etc/unbound/unbound.conf.d`. In this scenario we will be using Pihole in docker and Unbound running on bare system so configuration will be bit different from default one.

```
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: IP_ADDRESS_HOST
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to no if you don't have IPv6 connectivity
    do-ip6: no

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # IP fragmentation is unreliable on the Internet today, and can cause
    # transmission failures when large DNS messages are sent via UDP. Even
    # when fragmentation does work, it may not be secure; it is theoretically
    # possible to spoof parts of a fragmented DNS message, without easy
    # detection at the receiving end. Recently, there was an excellent study
    # >>> Defragmenting DNS - Determining the optimal maximum UDP response size for DNS <<<
    # by Axel Koolhaas, and Tjeerd Slokker (https://indico.dns-oarc.net/event/36/contributions/776/)
    # in collaboration with NLnet Labs explored DNS using real world data from the
    # the RIPE Atlas probes and the researchers suggested different values for
    # IPv4 and IPv6 and in different scenarios. They advise that servers should
    # be configured to limit DNS messages sent over UDP to a size that will not
    # trigger fragmentation on typical network links. DNS servers can switch
    # from UDP to TCP when a DNS response is too big to fit in this limited
    # buffer size. This value has also been suggested in DNS Flag Day 2020.
    edns-buffer-size: 1232

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement >
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10

    # Ensure no reverse queries to non-public IP ranges (RFC6303 4.2)
    private-address: 192.0.2.0/24
    private-address: 198.51.100.0/24
    private-address: 203.0.113.0/24
    private-address: 255.255.255.255/32
    private-address: 2001:db8::/32

    access-control: 127.0.0.0/8 allow
    access-control: 192.168.x.0/24 allow
    access-control: 172.16.0.0/8 allow
```
In field `IP_ADDRESS_HOST` we will put ip address of the machine where we installed unbound.

```
    access-control: 127.0.0.0/8 allow
    access-control: 192.168.x.0/24 allow
    access-control: 172.16.0.0/8 allow
```
This part is used for which ip address range can access dns queries. Ideally you should only change second line to match ip adresses on your network by changing number where `x` is.

After configuration is done we need to restart unbound with `sudo systemctl restart unbound` and make it start on boot with `sudo systemctl enable unbound`.

## 🔁 Keepalived

In root of repository copy `keepalived.conf` file to the `/etc/keepalived/`. Keepalived is necessary if we use multiple Pihole instances. It uses virtual ip adress to combine these two instances and when one of them fails it transfers load to other one depending on priority we set in config. Depending on instance we can set one to be MASTER instance and other ones to be BACKUP.

```
vrrp_instance VI_1 {
  state MASTER_OR_BACKUP
  interface NETWORK_INTERFACE
  virtual_router_id 198
  advert_int 1
  unicast_src_ip IP_HOST
  unicast_peer {
    IP_BACKUP
  }
  priority 20
  authentication {
    auth_type PASS
    auth_pass YOUR_PASSWORD
  }
  virtual_ipaddress {
    VIRT_IP
  }
}
```
In field `MASTER_OR_BACKUP` we replace it with one of those 2 options in capital letter depending if our instance will be main one or backup.

Next `NETWORK_INTERFACE` should be replaced with name of main network interface for current machine. We can find this out by using command `ip a` and looking for interface with ip address of our machine.

`virtual_router_id` can be anything it just needs to be unique.
`IP_HOST` - replace with ip of current machine.
`IP_BACKUP` - replace with ip of the back up Pihole instance

## 🧱 Docker Compose

