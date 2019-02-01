# l2tp-ipsec-vpn-client

Configure Ubuntu/Debian VPN clients using the command line.

You need the following:

1. VPN Server Address
2. Pre Shared Key
3. Username
4. Password

## Install

Install the following packages:

    sudo apt-get update
    sudo apt-get install -y strongswan xl2tpd

## Configure StrongSwan

Edit ipsec.conf:

    sudo nano /etc/ipsec.conf

Paste the following and save the file (replace n.n.n.n with your VPN Server Address):

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

Paste the following and save the file (replace your_pre_shared_key with your PSK value):

    : PSK "your_pre_shared_key"

## Configure xl2tpd

Edit xl2tpd.conf:

    sudo nano /etc/xl2tpd/xl2tpd.conf

Paste the following and save the file (replace n.n.n.n with your VPN Server Address):

    [lac myVPN]
    ; set this to the ip address of your vpn server
    lns = n.n.n.n
    ppp debug = yes
    pppoptfile = /etc/ppp/options.l2tpd.client
    length bit = yes

Edit /etc/ppp/options.l2tpd.client:

    sudo nano /etc/ppp/options.l2tpd.client

Paste the following and save the file (replace your_user_name and your_password with your VPN credentials):

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

Start the ipsec and l2tp connection:

    sudo mkdir -p /var/run/xl2tpd
    sudo touch /var/run/xl2tpd/l2tp-control
    sudo service strongswan restart
    sudo service xl2tpd restart
    sudo service ipsec restart
    sleep 7
    sudo ipsec up L2TP-PSK
    sleep 7
    sudo bash -c 'echo "c myVPN" > /var/run/xl2tpd/l2tp-control'
    sleep 7
    ifconfig

Check the output. You should now see a new interface ppp0. Interface ppp0 is needed to continue to the next step.

## Route

Routing traffic to an IP address in your internal network. Replace x.x.x.x with the addres you wish to communicate with through the tunnel device:

    sudo ip route add x.x.x.x via $(ip address show dev ppp0 | grep -Po '(?<=peer )(\b([0-9]{1,3}\.){3}[0-9]{1,3}\b)') dev ppp0

## Test

The VPN connection is now complete. Verify that your traffic is being routed properly. Repalce x.x.x.x with the addres you wish to communicate with through the tunnel device:

    ping x.x.x.x

## Debugging

Check the logs:

    dmesg | less /var/log/xl2tpd.log

## References

- [ubergarm/l2tp-ipsec-vpn-client:strongswan](https://github.com/ubergarm/l2tp-ipsec-vpn-client/tree/strongswan)
- [Openswan L2TP/IPsec VPN client setup](https://wiki.archlinux.org/index.php/Openswan_L2TP/IPsec_VPN_client_setup)
- [hwdsl2/setup-ipsec-vpn](https://github.com/hwdsl2/setup-ipsec-vpn/blob/master/docs/clients.md#configure-linux-vpn-clients-using-the-command-line)
