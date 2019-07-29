# l2tp-ipsec-vpn-client

Configure a Linux VPN client using the command line.

You need the following:

1. VPN Server Address
2. Pre Shared Key
3. Username
4. Password

## Install

Install the following packages:

### Ubuntu & Debian

    sudo apt-get update
    sudo apt-get -y install strongswan xl2tpd

### CentOS & RHEL

    yum -y install epel-release
    yum --enablerepo=epel -y install strongswan xl2tpd

### Fedora

    yum -y install strongswan xl2tpd

## Configure StrongSwan

Edit ipsec.conf:

    sudo nano /etc/ipsec.conf

Replace the file content with the following and save the file (replace n.n.n.n with your VPN Server Address):

    config setup

    conn %default
        ikelifetime=60m
        keylife=20m
        rekeymargin=3m
        keyingtries=1
        keyexchange=ikev1
        authby=secret
        ike=aes128-sha1-modp1024,3des-sha1-modp1024!
        esp=aes128-sha1-modp1024,3des-sha1-modp1024!

    conn L2TP-PSK
        keyexchange=ikev1
        left=%defaultroute
        auto=add
        authby=secret
        type=transport
        leftprotoport=17/1701
        rightprotoport=17/1701
        # set this to the ip address of your vpn server
        right=n.n.n.n

Edit ipsec.secrets:

    sudo nano /etc/ipsec.secrets

Replace the file content with the following and save the file (replace your_pre_shared_key with your PSK value):

    : PSK "your_pre_shared_key"

Additionaly, run the following **only if you are using CentOS/RHEL or Fedora**:

    mv /etc/strongswan/ipsec.conf /etc/strongswan/ipsec.conf.old 2>/dev/null
    mv /etc/strongswan/ipsec.secrets /etc/strongswan/ipsec.secrets.old 2>/dev/null
    ln -s /etc/ipsec.conf /etc/strongswan/ipsec.conf
    ln -s /etc/ipsec.secrets /etc/strongswan/ipsec.secrets

## Configure xl2tpd

Edit xl2tpd.conf:

    sudo nano /etc/xl2tpd/xl2tpd.conf

Append the following to the file and save it (replace n.n.n.n with your VPN Server Address):

    [lac myVPN]
    ; set this to the ip address of your vpn server
    lns = n.n.n.n
    ppp debug = yes
    pppoptfile = /etc/ppp/options.l2tpd.client
    length bit = yes

Edit /etc/ppp/options.l2tpd.client:

    sudo nano /etc/ppp/options.l2tpd.client

Replace the file content with the following and save the file (replace your_user_name and your_password with your VPN credentials):

    ipcp-accept-local
    ipcp-accept-remote
    refuse-eap
    require-mschap-v2
    noccp
    noauth
    logfile /var/log/xl2tpd.log
    idle 1800
    mtu 1410
    mru 1410
    defaultroute
    usepeerdns
    debug
    connect-delay 5000
    name your_user_name
    password your_password

## Connect

Run the following command each time you can to start the ipsec and l2tp connection:

### Ubuntu & Debian
    sudo mkdir -p /var/run/xl2tpd
    sudo touch /var/run/xl2tpd/l2tp-control
    sudo service strongswan restart
    sudo service xl2tpd restart
    sudo service ipsec restart
    sleep 8
    sudo ipsec up L2TP-PSK
    sleep 8
    sudo bash -c 'echo "c myVPN" > /var/run/xl2tpd/l2tp-control'
    sleep 8
    ifconfig

### CentOS/RHEL & Fedora
    sudo mkdir -p /var/run/xl2tpd
    sudo touch /var/run/xl2tpd/l2tp-control
    sudo service strongswan restart
    sudo service xl2tpd restart
    sleep 8
    sudo strongswan up L2TP-PSK
    sleep 8
    sudo bash -c 'echo "c myVPN" > /var/run/xl2tpd/l2tp-control'
    sleep 8
    ifconfig

Check the output. You should now see a new interface ppp0. Interface ppp0 is needed to continue to the next step.

## Route

Routing traffic to an IP address in your internal network. Replace x.x.x.x with the addres you wish to communicate with through the tunnel device:

    sudo ip route add x.x.x.x via $(ip address show dev ppp0 | grep -Po '(?<=peer )(\b([0-9]{1,3}\.){3}[0-9]{1,3}\b)') dev ppp0

## Error: Unable to resolve host on EC2 instances

If you did run the `route` command on an EC2 instance and got the error `"unable to resolve host <ip-x-x-x-x>: Resource temporarily unavailable"`, do the following and then rerun the commands from the Connect and Router sections.

Copy the hostname, from the error message, which will contain the private IP address in the form `ip-x-x-x-x`. For instance `ip-172-31-26-197`

Open the hosts file

    sudo nano /etc/hosts

Add a new entry within the hosts file to include the hostname:

    172.31.29.26 ip-172-31-29-26

## Test

The VPN connection is now complete. Verify that your traffic is being routed properly. Repalce x.x.x.x with the addres you wish to communicate with through the tunnel device:

    ping x.x.x.x

## Disconnect

To disconnect run the following:

### Ubuntu & Debian

    sudo bash -c 'echo "d myVPN" > /var/run/xl2tpd/l2tp-control'
    ipsec down L2TP-PSK

### CentOS/RHEL & Fedora

    sudo bash -c 'echo "d myVPN" > /var/run/xl2tpd/l2tp-control'
    sudo strongswan down L2TP-PSK

## Debugging

Check the logs:

    dmesg | less /var/log/xl2tpd.log

## References

- [ubergarm/l2tp-ipsec-vpn-client:strongswan](https://github.com/ubergarm/l2tp-ipsec-vpn-client/tree/strongswan)
- [Openswan L2TP/IPsec VPN client setup](https://wiki.archlinux.org/index.php/Openswan_L2TP/IPsec_VPN_client_setup)
- [hwdsl2/setup-ipsec-vpn](https://github.com/hwdsl2/setup-ipsec-vpn/blob/master/docs/clients.md#configure-linux-vpn-clients-using-the-command-line)
