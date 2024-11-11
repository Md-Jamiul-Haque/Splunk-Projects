# Creating a SIEM Environment Using Splunk & Sysmon
<div align="justify">
In this guide, we’ll dive into building a Security Information and Event Management (SIEM) setup with Splunk in a home lab environment—a valuable hands-on experience for anyone looking to sharpen their cybersecurity skills. Picture this: catching potential threats, analyzing log data in real time, and observing simulated attacks as they unfold. By the end of this walkthrough, you’ll have a fully functioning SIEM lab that forwards logs from two Windows hosts to a centralized Splunk server for analysis and monitoring.

For detailed logging on each Windows machine, we’ll use Sysmon, with Splunk Universal Forwarder handling log forwarding to the Splunk server. To see our detection capabilities in action, we’ll simulate attack scenarios using Atomic Red Team and also employ a Kali Linux machine to perform a brute-force attempt on a domain user account, testing our ability to catch unauthorized access attempts. This setup allows us to capture and analyze a variety of real-world attack patterns, all monitored through Splunk.

While these two Windows machines act as a domain controller and a domain user, we’ll keep the focus on the SIEM setup and won’t cover the Active Directory setup for now. Ready to get started? Let’s jump into the setup and bring your security lab to life.
</div>

# Diagram
<div>
  <img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/SIEM.drawio.svg" />
</div>

# Installing Ubuntu Server 24.04.01 LTS on Hyper-V
<div align="justify">
To kick off our SIEM setup, we’ll start by creating a virtual Ubuntu Server 24.04.01 LTS on Hyper-V. This will be the core machine running Splunk, where all our logs will be collected, stored, and analyzed. If you've worked with Hyper-V before, these steps should feel familiar, but we'll go through each in a clear, straightforward way to make sure no detail is missed.

### Step 1: Set Up the Virtual Machine
1. **Open Hyper-V Manager** on your host machine. From the Actions panel, select *New > Virtual Machine* to launch the New Virtual Machine Wizard.
2. **Name Your VM** – Give it a clear, descriptive name, like "Splunk-Server". Choose a location for saving the VM files if you want to keep things organized.
3. **Choose Generation** – Select *Generation 1* to ensure broad compatibility with Ubuntu Server.
4. **Assign Memory** – Set at least **2GB of memory** (2048 MB), but if your machine allows, consider allocating more for smoother operation.

### Step 2: Configure Networking
1. **Network Configuration** – Choose a virtual switch from the dropdown. If you haven’t created one, you can do so by going to *Virtual Switch Manager* in Hyper-V and creating an External Virtual Switch, which lets the VM access your network.

### Step 3: Set Up Virtual Disk
1. **Configure Disk Space** – Ubuntu Server doesn’t take up much room on its own, but since this VM will be handling logs, allocate at least **20GB of storage** to avoid running out of space later.

### Step 4: Attach the Ubuntu ISO
1. **Attach ISO** – Download the Ubuntu Server 24.04.01 LTS ISO if you haven’t already. Back in the VM setup wizard, select the option to *Install an operating system from a bootable image file* and browse for the ISO file you downloaded. This will allow the VM to boot directly into the Ubuntu installer.

### Step 5: Start the VM and Install Ubuntu
1. **Launch the VM** – After finalizing the setup, select *Connect* to open the VM window and then *Start* to power it on. You’ll see the Ubuntu installation begin.
2. **Follow the Installer Prompts** – Ubuntu’s installation process is straightforward:
   - **Language & Region**: Choose your preferred language and location.
   - **Keyboard Layout**: Confirm your keyboard layout.
   - **Network Configuration**: We'll update network configuration file later, so you can leave as it is.
   - **Configure Storage**: Use the entire virtual disk for Ubuntu (automatic partitioning is usually fine for this project).
   - **Set Up a User**: Create a username and password, and make sure to remember these—they’ll be your main credentials for the server.

### Step 6: Finalize Installation
1. **Complete and Reboot** – Once you’ve followed all prompts, Ubuntu will begin installing. This may take a few minutes. When it’s done, the system will prompt you to reboot.
2. **Remove the ISO** – Before rebooting, detach the ISO from the VM settings in Hyper-V to avoid restarting into the installer again.

### Step 7: First Login and Updates
1. **Log In** – Use the username and password you created.
2. **Update Ubuntu** – Run the following commands to bring everything up to date:
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```
### Step 8: Set a Static IP for the Splunk Server

From our setup diagram, we've chosen the static IP `10.27.221.250` for our Splunk server. To configure this IP, we'll need to set up a static IP on our Splunk server.

1. **Edit the Network Configuration File** – Open the network configuration file with the following command:

    ```bash
    sudo nano /etc/netplan/50-cloud-init.yaml
    ```

2. **Update the Configuration** – Modify the configuration file to reflect the following settings:

    ```yaml
    network:
      version: 2
      rendered: networkd
      ethernets:
        eth0:      # Replace with your network interface name
          dhcp4: false
          addresses:
            - 10.27.221.250/24  # Replace with your chosen Static Ip
          routes:
            - to: default
              via: 10.27.221.3  # Replace with your default gateway
          nameservers:
            addresses:
              - 8.8.8.8
    ```

   - **Note**: Replace `eth0` with the actual name of your network interface (you can find this by running `ip a`).
   - Replace `10.27.221.3` with your default gateway address.
   - Replace `10.27.221.250` with your static IP.

3. **Apply the Changes** – Once you've updated the configuration, save the file and apply the changes with:

    ```bash
    sudo netplan apply
    ```

This configuration assigns a static IP to the Splunk server, ensuring it matches our lab setup requirements. You can confirm this by running `ip a`.

</div>

# Installing Kali Linux on Hyper-V Using a Prebuilt Virtual Machine
<div align="justify">
In this section, we’ll set up Kali Linux on Hyper-V by downloading and using a prebuilt virtual machine image specifically designed for Hyper-V. This approach saves time and ensures we get a stable, optimized installation for our attack simulations.

### Step 1: Download the Prebuilt Kali Linux VM
1. **Visit the Official Kali Linux Website** – Go to [Kali Linux Downloads](https://www.kali.org/get-kali/) and scroll down to the *Virtual Machines* section.
2. **Choose the Hyper-V Version** – Select the *Hyper-V* option for your platform and download the prebuilt virtual machine image, which will be in the `.vhdx` format.

### Step 2: Extract the VM Files
1. **Extract the Downloaded File** – The `.vhdx` file will typically be inside a compressed `.zip` file. Use a tool like **7-Zip** or **WinRAR** to extract the `.vhdx` file to your desired location on your system.

### Step 3: Create a New Virtual Machine in Hyper-V
1. **Open Hyper-V Manager** – Launch the *Hyper-V Manager* on your host machine.
2. **Create a New VM** – From the Actions panel, click *New > Virtual Machine* to start the New Virtual Machine Wizard.
3. **Name Your VM** – Give your VM a descriptive name, like "Kali-Linux-VM".
4. **Choose Generation** – Select *Generation 1* to ensure compatibility with the `.vhdx` file.
5. **Assign Memory** – Allocate at least **2GB of RAM** (2048 MB) for Kali Linux, though you can assign more for better performance.
6. **Configure Networking** – Select a virtual switch to allow Kali Linux to access the network. This must be the same switch used by your Ubuntu server, enabling network access for attack simulations.

### Step 4: Attach the Prebuilt Kali Linux VHDX
1. **Use an Existing Virtual Hard Disk** – In the *Virtual Hard Disk* section of the wizard, select *Use an existing virtual hard disk*.
2. **Browse and Attach the `.vhdx` File** – Click *Browse* and navigate to the location where you extracted the `.vhdx` file. Select it and click *Open* to attach it to the VM.
3. **Complete the Setup** – Review your settings, and click *Finish* to create the VM.

### Step 6: Start Kali Linux VM
1. **Start the VM** – In Hyper-V Manager, select your new Kali Linux VM and click *Start*.
2. **Login to Kali Linux** – Once Kali boots up, log in with the default credentials:
   - **Username**: `kali`
   - **Password**: `kali`

### Step 7: Update Kali Linux
1. **Update Kali** – It’s a good idea to make sure your Kali Linux system is up to date. Open a terminal and run the following commands:
   ```bash
   sudo apt update && sudo apt upgrade -y
</div>

# Installing Splunk on Ubuntu Server
<div align="justify">
Now that we have our Ubuntu Server up and running, it's time to install Splunk. In this section, we'll walk you through the process of installing the free 14-day trial version of Splunk on our Ubuntu server.

### Step 1: Download Splunk
1. **Log in to Splunk Website** – On your host machine, open a web browser and go to the [Splunk Downloads Page](https://www.splunk.com/en_us/download.html).
2. **Choose the Version** – Select the *Splunk Enterprise* version and choose the *Linux (deb)* format.
3. **Copy the Download Link** – After selecting the desired version, right-click on the *Download Now* button and copy the wget download link.

Run the following command on your Ubuntu Server to download the package:
```bash
   [copy-the-link-here]
```
### Step 2: Install Splunk

1.  **Install the Splunk Package** – Once the download is complete, install Splunk using the following command:

    ```bash
    sudo dpkg -i splunk-8.3.0-c7.x86_64.deb
    ```

2.  **Accept the License** – During the installation, you'll be prompted to accept the license agreement. Type `y` and press Enter to accept it.

3.  **Start Splunk** – After the installation is complete, you can start Splunk with the following command:

    ```bash
    sudo /opt/splunk/bin/splunk start
    ```

    You will be prompted to set up the Splunk admin username and password during the first startup.

### Step 3: Enable Auto Start on Boot

To ensure Splunk starts automatically after the server is rebooted, run the following command:

```bash
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
```

</div>
