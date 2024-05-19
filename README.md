# PiNAS Firewall Project: Raspberry Pi Network-Attached Storage and Firewall Setup

This guide provides detailed instructions for setting up a 1TB HDD(it's not an SSD I know...it's what I had!) as a shared network folder on a Raspberry Pi and configuring a firewall using UFW. It includes optional steps for installing and configuring PiVPN and Pi-hole, along with troubleshooting tips for common issues.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Formatting the HDD](#formatting-the-hdd)
4. [Mounting the HDD](#mounting-the-hdd)
5. [Installing and Configuring Samba](#installing-and-configuring-samba)
6. [Setting Folder Permissions](#setting-folder-permissions)
7. [Accessing the Shared Folder](#accessing-the-shared-folder)
    - [From Windows](#from-windows)
    - [From Mac](#from-mac)
    - [From iPhone](#from-iphone)
8. [Configuring UFW](#configuring-ufw)
9. [Optional: Setting Up PiVPN](#optional-setting-up-pivpn)
10. [Optional: Setting Up Pi-hole](#optional-setting-up-pi-hole)
11. [Troubleshooting](#troubleshooting)
    - [Common Issues](#common-issues)
    - [Detailed Troubleshooting Steps](#detailed-troubleshooting-steps)
12. [Summary](#summary)

## Prerequisites

- A Raspberry Pi with Raspbian installed and configured.
- SSH access to the Raspberry Pi.
- A 1TB HDD connected to the Raspberry Pi.
- A Samba username and password (replace `<username>` with your preferred username).

## Initial Setup

1. **Connect HDD to Raspberry Pi**:
   - **Why?**: The 1TB HDD will be used as the storage device for your shared folder.
   - **How?**: Plug the 1TB HDD into a USB port on your Raspberry Pi.

2. **SSH into Raspberry Pi**:
   - **Why?**: SSH allows you to remotely manage and configure your Raspberry Pi.
   - **How?**:
     - Open your SSH client (like PuTTY) on your Windows or Mac device.
     - Connect to your Raspberry Pi using its IP address and your credentials.

## Formatting the HDD

1. **Identify the HDD**:
   - **Why?**: To know the device name of your HDD, which is necessary for formatting.
   - **How?**:
     ```bash
     lsblk
     ```
   - Identify your HDD (likely listed as `/dev/sda`).

2. **Unmount the HDD (if necessary)**:
   - **Why?**: Ensure the HDD is not in use before formatting to avoid data corruption.
   - **How?**:
     ```bash
     sudo umount /dev/sda1
     ```

3. **Create a New Partition Table**:
   - **Why?**: To prepare the HDD for a new file system.
   - **How?**:
     - Install `parted` if it’s not already installed:
       ```bash
       sudo apt-get install parted
       ```
     - Create a new partition table:
       ```bash
       sudo parted /dev/sda --script mklabel gpt
       ```
     - Create a new partition:
       ```bash
       sudo parted /dev/sda --script mkpart primary ext4 0% 100%
       ```

4. **Format the Partition**:
   - **Why?**: Formatting sets up a file system on the partition so it can store files.
   - **How?**:
     ```bash
     sudo mkfs.ext4 /dev/sda1
     ```

## Mounting the HDD

1. **Create a Mount Point**:
   - **Why?**: A mount point is a directory where the HDD's file system will be attached.
   - **How?**:
     ```bash
     sudo mkdir /mnt/myhdd
     ```

2. **Mount the HDD**:
   - **Why?**: Mounting makes the HDD’s file system accessible to the Raspberry Pi.
   - **How?**:
     ```bash
     sudo mount /dev/sda1 /mnt/myhdd
     ```

3. **Auto-Mount on Boot**:
   - **Why?**: Ensures the HDD is automatically mounted every time the Raspberry Pi boots.
   - **How?**:
     - Open the `fstab` file to configure auto-mounting:
       ```bash
       sudo nano /etc/fstab
       ```
     - Add the following line to ensure the HDD mounts on boot:
       ```bash
       /dev/sda1 /mnt/myhdd ext4 defaults,acl 0 2
       ```
     - Apply the new configuration:
       ```bash
       sudo mount -a
       ```

## Installing and Configuring Samba

1. **Install Samba**:
   - **Why?**: Samba allows file sharing between Linux and Windows systems.
   - **How?**:
     ```bash
     sudo apt-get update
     sudo apt-get install samba samba-common-bin
     ```

2. **Configure Samba**:
   - **Why?**: To set up the shared folder and control access permissions.
   - **How?**:
     - Open the Samba configuration file:
       ```bash
       sudo nano /etc/samba/smb.conf
       ```
     - Add the following at the end of the file to configure the shared folder:
       ```ini
       [Shared]
       comment = Shared Folder on HDD
       path = /mnt/myhdd/shared
       browseable = yes
       writable = yes
       only guest = no
       create mask = 0777
       directory mask = 0777
       public = no
       valid users = <username>
       vfs objects = acl_xattr
       map acl inherit = yes
       store dos attributes = yes
       ```

3. **Add Samba User**:
   - **Why?**: To create a user account for accessing the shared folder.
   - **How?**:
     ```bash
     sudo smbpasswd -a <username>
     ```

4. **Restart Samba Service**:
   - **Why?**: To apply the new configuration changes.
   - **How?**:
     ```bash
     sudo systemctl restart smbd
     ```

## Setting Folder Permissions

1. **Create and Set Permissions**:
   - **Why?**: Ensures that the user `<username>` has full access to the shared folder.
   - **How?**:
     - Create the shared folder:
       ```bash
       sudo mkdir /mnt/myhdd/shared
       ```
     - Set the appropriate permissions:
       ```bash
       sudo chmod -R 777 /mnt/myhdd/shared
       sudo chown -R <username>:<username> /mnt/myhdd/shared
       sudo setfacl -R -m u:<username>:rwx /mnt/myhdd/shared
       sudo setfacl -dR -m u:<username>:rwx /mnt/myhdd/shared
       ```

## Accessing the Shared Folder

### From Windows

1. **Open File Explorer**:
   - **Why?**: To connect to the shared folder.
   - **How?**: In the address bar, type the following, replacing `<Raspberry_Pi_IP_Address>` with the actual IP address of your Raspberry Pi:
     ```txt
     \\<Raspberry_Pi_IP_Address>\Shared
     ```

2. **Authenticate**:
   - **Why?**: To log in with the Samba username and password.
   - **How?**: Enter the Samba username (`<username>`) and password when prompted.

### From Mac

1. **Open Finder**:
   - **Why?**: To connect to the shared folder.
   - **How?**: Select **Go > Connect to Server** from the menu.

2. **Enter Server Address**:
   - **Why?**: To specify the location of the shared folder.
   - **How?**: Enter the following address, replacing `<Raspberry_Pi_IP_Address>` with the actual IP address of your Raspberry Pi:
     ```txt
     smb://<Raspberry_Pi_IP_Address>/Shared
     ```

3. **Authenticate**:
   - **Why?**: To log in with the Samba username and password.
   - **How?**: Enter the Samba username (`<username>`) and password when prompted.

### From iPhone

1. **Open Files App**:
   - **Why?**: To connect to the shared folder.
   - **How?**: Tap on the **Browse** tab at the bottom of the screen. Tap the **...** (ellipsis) in the upper right corner and select **Connect to Server**.

2. **Enter Server Address**:
   - **Why?**: To specify the location of the shared folder.
   - **How?**: In the "Connect to Server" dialog, enter the following address, replacing `<Raspberry_Pi_IP_Address>` with the actual IP address of your Raspberry Pi:
     ```txt
     smb://<Raspberry_Pi_IP_Address>/Shared
     ```

3. **Authenticate**:
   - **Why?**: To log in with the Samba username and password.
   - **How?**: Enter the Samba username (`<username>`) and password when prompted.

4. **Save Photos**:
   - **Why?**: To ensure compatibility with the file system.
   - **How?**: When attempting to upload images, change the format to "Most Compatible". This converts images to JPEG, which is widely supported.

## Configuring UFW

1. **Install UFW**:
   - **Why?**: UFW (Uncomplicated Firewall) helps secure your Raspberry Pi by controlling incoming and outgoing traffic.
   - **How?**:
     ```bash
     sudo apt-get install ufw
     ```

2. **Set Default Policies**:
   - **Why?**: By default, deny all incoming traffic and allow outgoing traffic to minimize security risks.
   - **How?**:
     ```bash
     sudo ufw default deny incoming
     sudo ufw default allow outgoing
     ```

3. **Allow Specific Services**:
   - **Why?**: Allow necessary traffic for SSH, PiVPN, Pi-hole, and local network access.
   - **How?**:
     - Allow SSH from trusted IP addresses:
       ```bash
       sudo ufw allow from <trusted_ip_address> to any port 22
       ```
     - Allow OpenVPN (PiVPN) on UDP port 1194:
       ```bash
       sudo ufw allow 1194/udp
       ```
     - Allow HTTP and HTTPS for Pi-hole:
       ```bash
       sudo ufw allow 80/tcp
       sudo ufw allow 443/tcp
       ```
     - Allow DNS for Pi-hole:
       ```bash
       sudo ufw allow 53/tcp
       sudo ufw allow 53/udp
       ```
     - Allow local network access:
       ```bash
       sudo ufw allow from 192.168.4.0/22
       ```

4. **Deny Unnecessary Traffic**:
   - **Why?**: Block all other ports to secure the system.
   - **How?**:
     ```bash
     sudo ufw deny 1:65535/tcp
     sudo ufw deny 1:65535/udp
     ```

5. **Enable UFW**:
   - **Why?**: Apply the firewall rules.
   - **How?**:
     ```bash
     sudo ufw enable
     ```

6. **Check UFW Status**:
   - **Why?**: Verify that the firewall is running and the rules are correctly applied.
   - **How?**:
     ```bash
     sudo ufw status verbose
     ```

## Optional: Setting Up PiVPN

1. **Install PiVPN**:
   - **Why?**: PiVPN allows you to set up a VPN server on your Raspberry Pi, providing secure remote access to your network.
   - **How?**:
     ```bash
     curl -L https://install.pivpn.io | bash
     ```
   - Follow the on-screen instructions to complete the installation.

2. **Configure PiVPN**:
   - **Why?**: To ensure PiVPN is set up correctly with the desired settings.
   - **How?**: Follow the prompts during installation to configure the VPN server.

3. **Allow OpenVPN(or WireGuard if not using OpenVPN) Traffic through UFW**:
   - **Why?**: Ensure that VPN traffic is not blocked by the firewall.
   - **How?**:
     ```bash
     sudo ufw allow 1194/udp
     ```
     or
     ```bash
     sudo ufw allow 51820/udp
     ```

## Optional: Setting Up Pi-hole

1. **Install Pi-hole**:
   - **Why?**: Pi-hole is a network-wide ad blocker that helps improve browsing performance and privacy.
   - **How?**:
     ```bash
     curl -sSL https://install.pi-hole.net | bash
     ```
   - Follow the on-screen instructions to complete the installation.

2. **Configure Pi-hole**:
   - **Why?**: To ensure Pi-hole is set up correctly and managing DNS requests for your network.
   - **How?**: Follow the prompts during installation to configure Pi-hole.

3. **Allow Pi-hole Traffic through UFW**:
   - **Why?**: Ensure that DNS and web interface traffic is not blocked by the firewall.
   - **How?**:
     ```bash
     sudo ufw allow 80/tcp
     sudo ufw allow 443/tcp
     sudo ufw allow 53/tcp
     sudo ufw allow 53/udp
     ```

## Troubleshooting

### Common Issues

- **"Operation couldn't be completed: attribute not found"**: This error occurs with certain image types. It indicates a problem with file attributes or permissions.
- **Permission Denied**: Unable to write to the shared folder. This usually happens due to incorrect folder permissions or Samba configuration.
- **Connectivity Issues**: Unable to connect to the shared folder from different devices. This can be due to network issues or incorrect Samba configuration.

### Detailed Troubleshooting Steps

1. **Ensure Full Permissions**:
   - **Why?**: Full permissions ensure that the Samba user `<username>` has complete access to the shared folder.
   - **How?**:
     ```bash
     sudo chmod -R 777 /mnt/myhdd/shared
     sudo chown -R <username>:<username> /mnt/myhdd/shared
     sudo setfacl -R -m u:<username>:rwx /mnt/myhdd/shared
     sudo setfacl -dR -m u:<username>:rwx /mnt/myhdd/shared
     ```

2. **Check File System Compatibility**:
   - **Why?**: The file system must support ACLs (Access Control Lists) to handle file attributes correctly.
   - **How?**:
     - Verify the HDD is formatted with `ext4` and supports ACL:
       ```bash
       sudo tune2fs -l /dev/sda1 | grep "Default mount options"
       ```
     - Remount the file system with ACL support:
       ```bash
       sudo umount /dev/sda1
       sudo mount -o acl /dev/sda1 /mnt/myhdd
       ```
     - Ensure `/etc/fstab` is configured for ACL:
       ```bash
       sudo nano /etc/fstab
       ```
       Add or modify the relevant line:
       ```ini
       /dev/sda1 /mnt/myhdd ext4 defaults,acl 0 2
       ```
       Remount the file system:
       ```bash
       sudo mount -a
       ```

3. **Adjust Samba Configuration**:
   - **Why?**: To ensure Samba supports extended attributes and logs detailed information for troubleshooting.
   - **How?**:
     - Open the Samba configuration file:
       ```bash
       sudo nano /etc/samba/smb.conf
       ```
     - Ensure the configuration includes:
       ```ini
       [global]
       log file = /var/log/samba/log.%m
       max log size = 1000
       log level = 3

       [Shared]
       comment = Shared Folder on HDD
       path = /mnt/myhdd/shared
       browseable = yes
       writable = yes
       only guest = no
       create mask = 0777
       directory mask = 0777
       public = no
       valid users = <username>
       vfs objects = acl_xattr
       map acl inherit = yes
       store dos attributes = yes
       ```
     - Restart the Samba service:
       ```bash
       sudo systemctl restart smbd
       ```

4. **Check Logs**:
   - **Why?**: Logs provide detailed information about errors and issues.
   - **How?**:
     - Monitor Samba logs for errors:
       ```bash
       sudo tail -f /var/log/samba/log.smbd
       ```
     - Check system logs for related errors:
       ```bash
       sudo dmesg | grep -i ext4
       sudo journalctl -xe
       ```

5. **Format Compatibility on iPhone**:
   - **Why?**: Changing the image format to "Most Compatible" ensures the images are saved in a widely supported format like JPEG.
   - **How?**: When attempting to upload images, select "Most Compatible" format before uploading.

## Summary

This guide provides comprehensive steps to set up and troubleshoot a network shared folder on a Raspberry Pi, accessible from Windows, Mac, and iPhone devices. It also covers configuring UFW to secure your system by controlling incoming and outgoing traffic. Optional configurations for PiVPN and Pi-hole are also included to enhance functionality. Ensure correct formatting, permissions, and Samba configuration to avoid common issues.

By following this guide, you should be able to successfully set up and maintain a network shared folder on your Raspberry Pi while ensuring it is secure with a properly configured firewall. If you encounter specific error messages or issues, refer to the troubleshooting section for detailed solutions.
