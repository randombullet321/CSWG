# CSWG ZeroTier Configuration for beginners

## Prerequisites

I am currently using IP space 172.25.0.0/16 and 192.168.175.0/24-192.168.200.0/24. Please let me know if this interferes with your current network. If it does you can try NATing behind your network.

I am currently running this on Proxmox with an isolated VLAN on my home network.

Current hardware is 2 vCPUs (Ryzen 5700G), 1.5GB RAM, 16GB SSD, and 1GBe NIC

OS is Ubuntu 22.04.1 LTS.

I am creating another VM that displays a webpage so you know that the VPN is working correctly. It's IP address is 10.0.1.8.

### Generating SSH keys (Mandatory)

Download [PuTTY](https://putty.org/).

Then "Run PuTTYgen"

![image](https://user-images.githubusercontent.com/94878618/215215239-b7a6aa13-3bff-4a82-a623-dd73c66e76d6.png)

On the bottom right click on Ed25519

![image](https://user-images.githubusercontent.com/94878618/215215338-b7ade2be-5065-45a7-aea0-9cb27445aafc.png)

Click Generate and follow it's instructions

On key comment please use your [ISO 3166 Alpha-3 country code](https://www.iban.com/country-codes) and your LAST.FIRST INITIAL

For example John Doe from the USA would be "USA.DOE.J"

Finally create strong password for your passphrase. If you lose this password, you will no longer be able to access your server.

Finally click "Save private key" I would save it with the same nomenclature as your key comment but change . to _ (USA_WU_A) so your computer won't think it's a .a file.

![image](https://user-images.githubusercontent.com/94878618/215216361-7842cd59-50d6-4a2c-99d7-fc59a4f0d89f.png)

Do not lose your key or your key passphrase as you will lose access to your server if you do.

## Starting up the server (Mandatory)

Go through all of your setup steps until you reboot. Please use SSH keys as this is an untrusted network. Assume this instance is open to the entire world.

If you do not have a chance to import your SSH keys since you don't have a Github or a Launchpad account, you can manually add them below.

### How to use an SSH key and disable password authentication (Mandatory)

1. Locate your SSH key and double click it. You will be prompted for your key passphrase. If you're not prompted, right click and "Load into Pageant"

2. SSH into your server using password authentication

3. Ensure your SSH key permissions are set on your server by pasting the codeblock below

        install -d -m 700 ~/.ssh
        
4. Locate your SSH key that you just created and right click it then "Edit with PuTTYgen" you will be prompted for your key passphrase.

5. Next copy your SSH public key

![image](https://user-images.githubusercontent.com/94878618/215216687-62c74356-6a4c-4169-91c9-74c8021c3a30.png)

6. Paste it into your authorized keys

        nano ~/.ssh/authorized_keys
        
7. Then edit your SSHD file to disable password authentication

        sudo nano /etc/ssh/sshd_config
        
8. Within the file find

        #    PasswordAuthentication yes
        
    Uncomment it and change yes to no

        PasswordAuthentication no
        
9. Finally restart sshd

        sudo systemctl restart sshd
        
10. Log out and try logging in with your key loaded into pageant.

You should be greeted with this.

![image](https://user-images.githubusercontent.com/94878618/215219984-5eb16edc-bc21-44b9-93ba-36d9dec32a71.png)

### A few guidelines beyond SSH security (Not mandatory)

I personally use UFW, but you can use IP Tables

1. Allow your SSH port first and only allow your trusted IPs (Management IP subnet)

2. Find the ports you will need to open. For this setup we will need 9993 UDP.

3. Finally remember to enable your firewall (sudo ufw Enable) and perhaps a service such as fail to ban.

Example UFW on my server

| To                             | Action | From          |
|--------------------------------|--------|---------------|
| 9993/udp                       | ALLOW  | Anywhere      |
| SSH/tcp                        | ALLOW  | MGMT Subnet   |
| Anywhere on ZeroTier interface | ALLOW  | 172.25.0.0/16 |
| 9993/udp (v6)                  | ALLOW  | Anywhere (v6) |

### General OS quality of life (Not mandatory)

First create a password for root

    sudo passwd root
    
Then login as root (This is not best practice, but it is good enough for our use case)

    su

Copy this codeblock so you can create an update script for root (Bypasses password, again not best practice but good enough for our use case)

    cat <<EOF > update.sh
    #!/bin/bash
    apt update && apt dist-upgrade -y
    apt autoremove -y
    apt autoclean && apt clean
    journalctl --vacuum-time=14d
    EOF

    chmod +x update.sh

    bash update.sh
    
Then updating it via Crontab

    # update using the update.sh file 03:00 every day
    0 3 * * * /root/update.sh

    # reboot every sunday at 00:00
    0 0 * * 0 /sbin/shutdown -r
    
## Actually installing ZeroTier (Everything here onwards is mandatory)

First run your update script by using

    bash update.sh
    
*If you’re willing to rely on SSL to authenticate the site, a one line install can be done with:*

    curl -s https://install.zerotier.com | sudo bash
        
*If you have GPG installed, a more secure option is available:*

    curl -s 'https://raw.githubusercontent.com/zerotier/ZeroTierOne/master/doc/contact%40zerotier.com.gpg' | gpg --import && \
    if z=$(curl -s 'https://install.zerotier.com/' | gpg); then echo "$z" | sudo bash; fi
        
## Joining the CSWG VPN Mesh

Here is where you actually join the VPN network (My email to you will have the network ID)

    sudo zerotier-cli join XXXXXXXXXXXXXXXX
        
You should see
   
    "200 join OK"
    
Next find out your ZeroTier ID using

    sudo zerotier-cli info
    
You should see 200 info XXXXXXXXXX 1.10.2 ONLINE. Please note your 10 digit identifier and shoot me an email of your identifier. I will need to approve you to join the network. Please ensure that I have your country and name within the email.

## Routing your LAN traffic through your ZeroTier instance.

Once I get your email, I will assigning you an IP space. Please ensure your network does not have this IP space already allocated.

Enable IP Forwarding for your server

    sudo sysctl -w net.ipv4.ip_forward=1
    
Now you will need to find the names of your interfaces

    ip a
    
You should see your ethernet interface and your ZeroTier interface (One starting with zt)

Create some shell variables to help with the next code (Modify your PHY_IFACE and ZT_IFACE to match your interfaces)

    PHY_IFACE=eth0
    ZT_IFACE=ztxxxxxxx
    
Next add your iptable rules

    sudo iptables -t nat -A POSTROUTING -o $PHY_IFACE -j MASQUERADE
    sudo iptables -A FORWARD -i $ZT_IFACE -o $PHY_IFACE -j ACCEPT
    sudo iptables -A FORWARD -i $PHY_IFACE -o $ZT_IFACE -m state --state RELATED,ESTABLISHED -j ACCEPT
    
Save the iptable rules on boot

    sudo apt install iptables-persistent
    sudo iptables-save | sudo tee /etc/iptables/rules.v4

Finally, make sure that it survives reboots

    sudo nano /etc/sysctl.conf
        
Go to where it says 

    # net.ipv4.ip_forward = 1
    
And uncomment it and make sure it says 1.

## Testing Part 1

Ping 192.168.175.99 via your server. You should get success.

## Testing Part 2

Please tell me your ZeroTier router IP address and I will create a static route within the VPN network.

Create another server in your LAN subnet (I'll use 192.168.176.3) and advertise your routes in your router. For example if I gave you subnet 192.168.176.0/24 and your ZeroTier server is 192.168.176.2, you should put a static route of 172.25.0.0/16 with a next hop of 192.168.176.2.

You server (192.168.176.3) should ping all the way through to my webserver (192.168.175.99)

Let me know which of your next address to ping and I should be able to ping from my webserver (192.168.175.99) to your other server (192.168.176.3)

If this works both ways, we have finally finished the VPN setup.

# Additional quality of life improvements

You can add a static route from your LAN devices to the ZeroTier instance. This will offload requests to your router and reduce a hop.
