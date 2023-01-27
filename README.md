# CSWG ZeroTier Configuration

I am currently running this on Proxmox with an isolated VLAN on my home network.

Current hardware is 2vCPUs (Ryzen 5700G), 1.5GB RAM, 16GB SSD, and 1GBe NIC

OS is Ubuntu 22.04.1 LTS.

I am creating another VM that displays a webpage so you know that the VPN is working correctly. It's IP address is 10.0.1.8.

    sudo nano /etc/ssh/sshd_config
    
Change #Port 22
