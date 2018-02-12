This bind9 server is going to act as an internal DNS server for me. It’s also going to forward queries that it is not authoritative for to Google’s DNS servers 8.8.8.8 and 8.8.4.4.

## Step 1: Install Bind9
`apt install bind9 -y`

## Step 2: Network Settings
#### Change to static IP addres if you use Debian as a Network Server.
`nano /etc/network/interfaces`

```
iface ens3 inet static
address 192.168.1.7
netmask 255.255.248.0
gateway 192.168.1.1
dns-nameservers 192.168.1.7
```
[ens3] is different on each environment, replace it to your own one.

I am giving my dns server ip address [192.168.1.7]

Replace [ens3], [address], [netmask], [gateway] and [dns-nameservers] to your own environment.

[dns-nameservers] is your dns server address.

#### Edit resolv.conf

`nano /etc/resolv.conf`

#### Add this line

`nameserver 192.168.1.7`

## Step 3: Loggin
First let me set up logging.  Out of the box on Ubuntu bind9 logs, via the syslog tool, to /var/log/syslog.    That’s fine and all, but I prefer to log to a standalone bind9 log file so I am going to set that up.

#### First edit /etc/bind/named.conf
`nano /etc/bind/named.conf`

#### Add the following line
`include "/etc/bind/named.conf.log";`

#### Then create the /etc/bind/named.conf.log file
`nano /etc/bind/named.conf.log`

#### And put the following in it
```
logging {
  channel bind_log {
    file "/var/log/bind/bind.log" versions 3 size 5m;
    severity info;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  category default { bind_log; };
  category update { bind_log; };
  category update-security { bind_log; };
  category security { bind_log; };
  category queries { bind_log; };
  category lame-servers { null; };
};
```
#### Run the following commands to create folders and set ownership
`mkdir /var/log/bind`

`chown bind:root /var/log/bind`

`chmod 775 /var/log/bind`

### Step 4: Tweak an apparmor setting.
Step3 is optional. If apparmor is not running on your system then no need to do this step. You could check apparmor status by run this command `service apparmor status`

`nano /etc/apparmor.d/usr.sbin.named`

#### Update this section (comment out /var/log/named and add /var/log/bind)
```
  # some people like to put logs in /var/log/named/ instead of having
  # syslog do the heavy lifting.
  #/var/log/named/** rw,
  #/var/log/named/ rw,
  /var/log/bind/** rw,
  /var/log/bind/ rw,
 ```
 
 #### Restart apparmor
 `service apparmor restart`
 
 ## Step 5: Bind Option
 
 
 ### Setup Forwarders
 #### Edit the /etc/bind/named.conf.options file.
 `nano /etc/bind/named.conf.options`
 
 #### Add following lines
 ```
 forward only;
 forwarders {
    8.8.8.8;
    8.8.4.4;
 };
 listen-on-v6 { none; };
```

#### Restart bind9
`service bind9 restart`

### Let’s test it. First I am going to tail the log file
`tail -f /var/log/bind/bind.log`

#### Then run this command to test it
My dns server has ip address 192.168.1.7 so I am going to use that with the dig command to get some results.

(dig @<your dns server ip> <destination ip address>)
  
`dig @192.168.1.7 www.google.com +short`
 
Now check the log. If its working log result will be like this.
>12-Feb-2018 13:52:50.951 queries: info: client 192.168.1.7#41168 (www.google.com): query: google.com IN A +E (192.168.1.7)

### Setup Zones
#### Edit /etc/bind/named.conf.local to set up zones for DNS this server will be serve.
`nano /etc/bind/named.conf.local`

##### Place the following in it.
```
zone "example.com" {
        type master;
        file "/etc/bind/db.example.com";
};

zone "168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.168.192";  # 192.168.0.0/16 subnet
};
```
Replace [168.192.in-addr.arpa] as per your network

This defines zones example.com  The file for each zone will define what to return when there is a DNS lookup for its zone.

Also this contains a reverse lookup 168.192.in-addr-arpa that will handle lookups of IP addresses and return hostnames.

#### Create the zones files
`nano /etc/bind/db.example.com`

##### Place the following in it
```
$TTL    86400
@       IN      SOA     ns1.example.com. admin.example.com. (
                        01012018        ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
; List Nameservers
                        IN      NS      ns1.example.com.
; Address to name mapping
ns1.example.com. IN      A       192.168.1.7
```

#### Create the zones files
`nano /etc/bind/db.168.192`

##### Place the following in it
```
$TTL    86400
@       IN      SOA     ns1.example.com. admin.example.com. (
                        20180101        ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
; Nameservers
        IN      NS      ns1.example.com.


; PTR Records
7.0     IN      PTR     ns1.example.com. ; 192.168.1.7
```
#### Run the following commands to set ownership

`chown -R bind:root /etc/bind/`

`chmod -R 755 /etc/bind/`

#### Restart bind9
Restart the bind9 service

`service bind9 restart`
