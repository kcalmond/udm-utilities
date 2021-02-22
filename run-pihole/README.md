# Run Pi-hole on your UDM

## Features

1. Run Pi-hole on your UDM with a completely isolated network stack.  This will not port conflict or be influenced by any changes on by Ubiquiti
2. Persists through reboots and firmware updates.

## Requirements

1. You have successfully setup the on boot script described [here](https://github.com/boostchicken/udm-utilities/tree/master/on-boot-script)

**MYCONFIG**: Also followed the next step in the top-level README to setup 05-container-common.sh 

## Customization

* Feel free to change [20-dns.conflist](../cni-plugins/20-dns.conflist) to change the IP address and MAC address of the container.
* Update [10-dns.sh](../dns-common/on_boot.d/10-dns.sh) with your own values
* If you want IPv6 support use [20-dnsipv6.conflist](../cni-plugins/20-dnsipv6.conflist) and update [10-dns.sh](../dns-common/on_boot.d/10-dns.sh) with the IPv6 addresses. Also, please provide IPv6 servers to podman using --dns arguments.

## Steps

1. On your controller, make a Corporate network with no DHCP server and give it a VLAN. For this example we are using VLAN 5.

    **MYCONFIG**: My UDMP is running a single default corp network, setup as 192.168.0.1/24. For this step I added a new corp network on VLAN=5 as 192.168.1.254/24; DHCP=none. Left all other settings at default.

2. Copy [20-dns.conflist](../cni-plugins/20-dns.conflist) to /mnt/data/podman/cni.  This will create your podman macvlan network

    **MYCONFIG**: copy my_20-dns.conflist to /mnt/data/podman/cni/20-dns.conflist

    Per [this comment](https://old.reddit.com/r/Ubiquiti/comments/lkwpox/following_boostchickens_awesome_config_for/gnnus3o/), I used https://macaddress.io/mac-address-generator to create a **registered** mac address for the "mac" value below:

    ```
    {
      "cniVersion": "0.4.0",
      "name": "dns",
      "plugins": [
        {
          "type": "macvlan",
          "mode": "bridge",
          "master": "br5",
          "mac": "24:bc:82:e7:07:d2",
          "ipam": {
            "type": "static",
            "addresses": [
              {
                "address": "192.168.1.15/24",
                "gateway": "192.168.1.1"
              }
            ],
            "routes": [
              {"dst": "0.0.0.0/0"}
            ]
          }
        }
      ]
    }
    ```


3. Copy [10-dns.sh](../dns-common/on_boot.d/10-dns.sh) to /mnt/data/on_boot.d and update its values to reflect your environment (**MYCONFIG**: copy my_10-dns.sh to /mnt/data/on_boot.d/10-dns.sh)

   ```
   ...
   VLAN=5
   IPV4_IP="192.168.1.15"
   IPV4_GW="192.168.1.1/24"
   ...
   CONTAINER=pihole
   ...
   ```   

4. Execute /mnt/data/on_boot.d/10-dns.sh
5. Create directories for persistent Pi-hole configuration

   ```
   mkdir -p /mnt/data/etc-pihole
   mkdir -p /mnt/data/pihole/etc-dnsmasq.d
   ```
   
6. Create and run the Pi-hole docker container. The following command sets the upstream DNS servers to 1.1.1.1 and 8.8.8.8.

    ```sh
     podman run -d --network dns --restart always \
        --name pihole \
        -e TZ="America/Los Angeles" \
        -v "/mnt/data/etc-pihole/:/etc/pihole/" \
        -v "/mnt/data/pihole/etc-dnsmasq.d/:/etc/dnsmasq.d/" \
        --dns=127.0.0.1 \
        --dns=1.1.1.1 \
        --dns=8.8.8.8 \
        --hostname pi.hole \
        -e VIRTUAL_HOST="pi.hole" \
        -e PROXY_LOCATION="pi.hole" \
        -e ServerIP="192.168.1.15" \
        -e IPv6="False" \
        pihole/pihole:latest
    ```
    Test customization for almond.lan:

    ```sh
    podman run -d --network dns --restart always \
       --name pihole \
       -e TZ="America/Los Angeles" \
       -v "/mnt/data/etc-pihole/:/etc/pihole/" \
       -v "/mnt/data/pihole/etc-dnsmasq.d/:/etc/dnsmasq.d/" \
       --dns=127.0.0.1 \
       --dns=1.1.1.1 \
       --dns=8.8.8.8 \
       --hostname udmpihole.almond.lan \
       -e VIRTUAL_HOST="udmpihole.almond.lan" \
       -e PROXY_LOCATION="udmpihole.almond.lan" \
       -e ServerIP="192.168.1.15" \
       -e IPv6="False" \
       pihole/pihole:latest
    ```
    Test customization for pihole+unbound using cbcrowe image:

    ```sh
    podman run -d --network dns --restart always \
    --name pihole \
    -e TZ="America/Los_Angeles" \
    -v "/mnt/data/etc-pihole/:/etc/pihole/" \
    -v "/mnt/data/pihole/etc-dnsmasq.d/:/etc/dnsmasq.d/" \
    -e DNS1=127.0.0.1#5335 \
    -e DNS2=127.0.0.1#5335 \
    -e DNSSEC=true \
    --hostname udmpihole.almond.lan \
    -e VIRTUAL_HOST="testpihole.almond.lan" \
    -e PROXY_LOCATION="testpihole.almond.lan" \
    -e REV_SERVER=true \
    -e REV_SERVER_DOMAIN="almond.lan" \
    -e REV_SERVER_TARGET="192.168.0.1" \
    -e REV_SERVER_CIDR="192.168.0.0/16" \
    cbcrowe/pihole-unbound:latest
    ```

    The below errors are expected and acceptable

    ```sh
    ERRO[0022] unable to get systemd connection to add healthchecks: dial unix /run/systemd/private: connect: no such file or directory
    ERRO[0022] unable to get systemd connection to start healthchecks: dial unix /run/systemd/private: connect: no such file or directory
    ```

6. Set pihole password

    ```sh
    podman exec -it pihole pihole -a -p YOURNEWPASSHERE
    ```

7. Update your DNS Servers to 192.168.1.15 for each of your Networks (UDM GUI | Networks | Advanced | DHCP Name Server)
8. Access the pihole like you would normally, e.g. http://192.168.1.15
