# CSWG ZeroTier Configuration for beginners

I am currently running this on Proxmox with an isolated VLAN on my home network.

Current hardware is 2 vCPUs (Ryzen 5700G), 1.5GB RAM, 16GB SSD, and 1GBe NIC

OS is Ubuntu 22.04.1 LTS.

I am creating another VM that displays a webpage so you know that the VPN is working correctly. It's IP address is 10.0.1.8.

# Creating SSH keys

Download [PuTTY](https://putty.org/).

Then "Run PuTTYgen"

![image](https://user-images.githubusercontent.com/94878618/215215239-b7a6aa13-3bff-4a82-a623-dd73c66e76d6.png)

On the bottom right click on Ed25519

![image](https://user-images.githubusercontent.com/94878618/215215338-b7ade2be-5065-45a7-aea0-9cb27445aafc.png)

Click Generate and follow it's instructions

On key comment please use your [ISO 3166 Alpha-3 country code](https://www.iban.com/country-codes) and your LAST.FIRST INITIAL

For example Albert Wu from the USA would be "USA.WU.A"

Finally create strong password for your passphrase. If you lose this password, you will no longer be able to access your server.

Finally click "Save private key" I would save it with the same nomencalture as your key comment but change . to _ (USA_WU_A) so your computer won't think it's a .a file.

![image](https://user-images.githubusercontent.com/94878618/215216361-7842cd59-50d6-4a2c-99d7-fc59a4f0d89f.png)

Do not lose your key or your key passphrase as you will lose access to your server if you do.

# Starting up the server

Go through all of your setup steps until you reboot. Please use SSH keys as this is an untrusted network. Assume this instance is open to the entire world.

If you do not have a chance to import your SSH keys since you don't have a Github or a Launchpad account, you can manually add them below.

# How to use an SSH key and disable password authentication

1. Locate your SSH key and double click it. You will be promted for your key passphrase. If you're not prompted, right click and "Load into Pageant"

2. SSH into your server using password authentication

3. Ensure your SSH key permissions are set on your server by pasting the codeblock below

        install -d -m 700 ~/.ssh
        
4. Locate your SSH key that you just created and right click it then "Edit with PuTTYgen" you will be promted for your key passphrase.

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

You should be greated with this.

A few guidelines beyond SSH security

1. Unblock your SSH port first and only allow your trusted IPs

2. Find the ports you will need to open. For this setup we will need 9993 UDP.

3. Finally remeber to enable your firewall and perhaps a service such as fail to ban.

Example UFW on my server

| To                             | Action | From          |
|--------------------------------|--------|---------------|
| 9993/udp                       | ALLOW  | Anywhere      |
| SSH/tcp                        | ALLOW  | MGMT Subnet   |
| Anywhere on ZeroTier interface | ALLOW  | 10.244.0.0/16 |
| 9993/udp (v6)                  | ALLOW  | Anywhere      |

# General OS quality of life

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
    
    
# Actually installing ZeroTier

First run your update script by using

    bash update.sh
    
