# DNS_Server
In this tutorial we will try to install BIND9 on DNS Server.
The address of DNS Server is 10.1.2.3
On DNS server install BIND with yum:

yum install -y bind bind-utils

Change the main configuration file of BIND named.conf:

vi /etc/named.conf
    //
    // named.conf
    //
    // Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
    // server as a caching only nameserver (as a localhost DNS resolver only).
    //
    // See /usr/share/doc/bind*/sample/ for example named configuration files.
    //
    // See the BIND Administrator's Reference Manual (ARM) for details about the
    // configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

    options {
        listen-on port 53 { any; };
    #   listen-on-v6 port 53 { none; };
        directory   "/var/named";
        dump-file   "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";

    /*
     - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
     - If you are building a RECURSIVE (caching) DNS server, you need to enable
       recursion.
     - If your recursive DNS server has a public IP address, you MUST enable access
       control to limit queries to your legitimate users. Failing to do so will
       cause your server to become part of large scale DNS amplification
       attacks. Implementing BCP38 within your network would greatly
       reduce such attack surface
    */

    recursion yes;
        forwarders {
            192.168.2.3;
            192.168.2.4;
        };
    forward only;
    dnssec-enable yes;
    dnssec-validation no;
    allow-recursion { any; };
    allow-query     { any; };
    allow-query-cache { any; };

    /* Path to ISC DLV key */
    bindkeys-file "/etc/named.iscdlv.key";

    managed-keys-directory "/var/named/dynamic";
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
    };

    logging {
        channel default_debug {
            file "data/named.run";
            severity dynamic;
        };
    };

    zone "." IN {
        type hint;
        file "named.ca";
    };

    include "/etc/named.rfc1912.zones";
    include "/etc/named.root.key";
    include "/etc/named/named.conf.local";

Edit named.conf.local file with our forward and reverse zones:

vi /etc/named/named.conf.local
    //
    // Do any local configuration here
    //

    // Consider adding the 1918 zones here, if they are not used in your
    // organization
    //include "/etc/bind/zones.rfc1918";

    zone "lv218.devops" {
        type master;
        file "/etc/named/zones/db.lv218.devops";
    };

    zone "1.10.in-addr.arpa" {
        type master;
        file "/etc/named/zones/db.10.1";
    };

Create Forward Zone File:

chmod 755 /etc/named
mkdir /etc/named/zones

vi /etc/named/zones/db.lv218.devops
    ;
    ; BIND data file for local loopback interface
    ;
    $TTL    604800
    @   IN  SOA     lv218.devops. root.lv218.devops. (
                          3         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )   ; Negative Cache TTL
    ;
                      IN      NS      lv218.devops.
    lv218.devops.     IN      A       10.1.2.3
    gitlab            IN      A       10.1.1.3
    www.gitlab        IN      A       10.1.1.3

Create Reverse Zone File:

vi /etc/named/zones/db.10.1
    ;
    ; BIND reverse data file for local loopback interface
    ;
    $TTL    604800
    @   IN  SOA        lv218.devops. root.lv218.devops. (
                          3         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )   ; Negative Cache TTL
    ;
            IN    NS      lv218.devops.
    3.2     IN    PTR     lv218.devops.
    3.1     IN    PTR     gitlab
    3.1     IN    PTR     www.gitlab

Check BIND Configuration Syntax:

named-checkconf
cd /etc/named/zones/
named-checkzone lv218.devops db.lv218.devops
    zone lv218.devops/IN: loaded serial 2016021201
    OK
named-checkzone 10.1.in-addr.arpa db.10.1
    zone 10.1.in-addr.arpa/IN: loaded serial 2016021201
    OK

Start BIND via systemctl:

systemctl start named
systemctl enable named

Configure DNS Clients:

vi /etc/resolv.conf
    nameserver 10.1.2.3
    nameserver 10.1.2.4

Test Clients

Forward Lookup
    nslookup lv218.devops
        Server:     10.1.2.3
        Address:    10.1.2.3#53

        Name:   lv218.devops
        Address: 10.1.2.3

Reverse Lookup
    nslookup 10.1.2.3
        Server:     10.1.2.3
        Address:    10.1.2.3#53

        3.2.1.10.in-addr.arpa   name = lv218.devops.
