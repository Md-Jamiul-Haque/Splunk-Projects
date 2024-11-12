# Creating a SIEM Environment Using Splunk & Sysmon
<div align="justify">
In this guide, we’ll dive into building a Security Information and Event Management (SIEM) setup with Splunk in a home lab environment—a valuable hands-on experience for anyone looking to sharpen their cybersecurity skills. Picture this: catching potential threats, analyzing log data in real time, and observing simulated attacks as they unfold. By the end of this walkthrough, you’ll have a fully functioning SIEM lab that forwards logs from two Windows hosts to a centralized Splunk server for analysis and monitoring.

For detailed logging on each Windows machine, we’ll use Sysmon, with Splunk Universal Forwarder handling log forwarding to the Splunk server. To see our detection capabilities in action, we’ll simulate attack scenarios using Atomic Red Team and also employ a Kali Linux machine to perform a brute-force attempt on a domain user account, testing our ability to catch unauthorized access attempts. This setup allows us to capture and analyze a variety of real-world attack patterns, all monitored through Splunk.

While these two Windows machines act as a domain controller and a domain user, we’ll keep the focus on the SIEM setup and won’t cover the Active Directory setup for now. Ready to get started? Let’s jump into the setup and bring your security lab to life.
</div>

# Diagram
<p align="center">
  <img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/SIEM.png" />
</p>

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
   - Please ensure the indentation is correct in the `50-cloud-init.yaml` file.
<p align="center">
<img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/netplan.png" />
<p></p>
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
   ```
### Step 8: Set a Static IP for Kali Linux Using the GUI

To configure a static IP for Kali Linux in our lab setup, follow these steps:

1. **Open Network Settings** – Go to the top-right corner of the Kali Linux desktop, click on the **Network** icon, and select **Settings**.

2. **Select the Network Interface** – In the **Settings** window, choose the network interface you want to configure (usually `Wired 1`).

3. **Set IP Address Method to Manual**:
   - In the interface settings, find the **IPv4** tab.
   - Change the **IPv4 Method** to **Manual**.

4. **Configure the Static IP Details**:
   - Under **Addresses**, add the following details:
     - **Address**: `10.27.221.242` (or any other IP within the same subnet, ensuring no IP conflicts).
     - **Netmask**: `255.255.255.0` (or `/24`).
     - **Gateway**: `10.27.221.3`  # replace with your network's default gateway.
   - In **DNS Servers**, add `8.8.8.8`. # you can use your preferred DNS server.

5. **Save and Apply**:
   - Click **Apply** to save the configuration.
   - Restart the network connection if needed, or simply disconnect and reconnect.

After completing these steps, your Kali Linux machine will have a static IP set in the network, matching our lab environment configuration.

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
   [copy-the-link-and-paste-it-in-your-Ubuntu-server]
```
### Step 2: Install Splunk

1.  **Install Splunk** – Use `dpkg` to install the Splunk `.deb` package:

    ```bash
    sudo dpkg -i splunk-8.3.0-c7.x86_64.deb
    ```

2. **Navigate to the Installation Directory** – Once installation is complete, go to the directory where Splunk is installed:

    ```bash
    cd /opt/splunk
    ls -la
    ```

    Here, you can see that all files belong to the `splunk` user and group.

3. **Switch to Splunk User** – Change to the `splunk` user to start Splunk under this user:

    ```bash
    sudo -u splunk bash
    ```

4. **Start Splunk** – Navigate to the Splunk `bin` directory and start Splunk:

    ```bash
    cd /opt/splunk/bin
    ./splunk start
    ```

5. **Accept the License Agreement and Set Up Admin Credentials** – You’ll be prompted to accept the license agreement. 

   - Type `Q` to scroll to the end of the agreement.
   - Type `y` and press Enter to accept.
   - After accepting the license, you will be prompted to create an admin `username` and `password`. Follow the prompts to set these credentials, which you will use to log into the Splunk web interface.


6. **Exit Splunk User** – After starting Splunk, exit the `splunk` user shell:

    ```bash
    exit
    ```

7. **Enable Splunk to Start on Boot** – Go back to the `bin` directory and enable Splunk to start automatically at boot, using the `splunk` user:

    ```bash
    cd /opt/splunk/bin
    sudo ./splunk enable boot-start -user splunk
    ```

This will ensure that Splunk starts automatically each time the server reboots, running under the `splunk` user.
<p align="center">
<img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/status.PNG" width="80%"/>
</p>
</div>

# Installing Sysmon with Olaf’s Configuration File on Windows

We’ll set up **Sysmon** (System Monitor), which captures detailed information about events like process creation, network connections, and file changes. We’ll use Olaf’s recommended configuration file for this. 

#### Step 1: Download Sysmon and Olaf’s Configuration File

1. **Download Sysmon** – Go to Microsoft’s [Sysinternals page](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) and download the latest version of Sysmon.
2. **Download Olaf’s Sysmon Configuration** – Visit [Olaf’s GitHub page](https://github.com/olafhartong/sysmon-modular) to get the configuration file:
   - Download the `sysmon-config.xml` file by saving its raw content as `sysmon-config.xml`.

#### Step 2: Install Sysmon with the Configuration File

1. To keep it simple, place `Sysmon.exe` and `sysmon-config.xml` files together.
2. Open **Command Prompt** with Administrator privileges.
3. Navigate to the Sysmon directory:

    ```bash
    cd C:\Users\User_name\Downloads\Sysmon
    ```

3. Run Sysmon with Olaf’s configuration file:

    ```bash
    .\Sysmon64.exe -i sysmon-config.xml
    ```

   - Accept the Sysmon license agreement.
   - The `-i` flag specifies the configuration file to use.

4. You’ll see output confirming that Sysmon has been successfully installed with the custom configuration.

#### Step 3: Verify Sysmon is Running

To make sure Sysmon is running:

1. Open **Event Viewer** by typing "Event Viewer" in the Windows search bar.
2. Navigate to **Applications and Services Logs > Microsoft > Windows > Sysmon**.
3. You should see events under the **Operational** log, indicating that Sysmon is actively recording events.

Sysmon is designed to automatically start at boot, so no additional configuration is required for this.

#### Step 4: Repeat for Additional Windows Machines

Follow these same steps on any additional Windows machines you’re monitoring. Each machine will now be configured to capture detailed event logs, ready to be forwarded to your Splunk server for analysis.

# Installing Splunk Universal Forwarder on Windows

To monitor Windows logs with Splunk, we need to install the **Splunk Universal Forwarder** on each Windows machine. This lightweight forwarder sends logs to our main Splunk instance for centralized analysis. Let's go through the process step-by-step, and be sure to follow these steps for each Windows machine in your environment.

### Step 1: Download the Splunk Universal Forwarder

1. On the Windows machine, open a web browser and go to the [Splunk Universal Forwarder download page](https://www.splunk.com/en_us/download/universal-forwarder.html).
2. Download the installer for Windows (usually a `.msi` file).

### Step 2: Run the Installer

1. Locate the downloaded `.msi` file and double-click it to start the installation.
2. Follow the installer prompts. Here are a few key settings:
   - **Installation Directory**: Choose the default or specify a custom path.
   - **Select Components**: Ensure that "Forwarder" is selected.
   - **Set Installation Type**: Choose "Local System".

### Step 3: Configure the Forwarder to Send Logs to the Splunk Server

1. When prompted, enter the **Splunk server’s IP address and receiving port**. This is where your Splunk server will receive data from the forwarder:
   - **Server IP**: `10.27.221.250` (replace it with your Splunk server IP)
   - **Port**: `9997` (make sure this matches the receiving port on your Splunk server)

2. **Set the Admin Credentials** – During setup, you’ll also be asked to set up an admin `username` and `password` for the forwarder. Create a username and password that you’ll remember, as these credentials may be required for managing the forwarder later.

Finish the installation and wait for it to complete. You’ll see a confirmation screen once it’s installed and ready.

### Step 4: Configuring the Splunk Universal Forwarder with `inputs.conf` to Send Logs to Splunk

Now that our Splunk Universal Forwarder (UF) is up and running, we need to specify which events we want to forward to our Splunk server. This is done by configuring an `inputs.conf` file, where we’ll outline the specific logs to capture, including **Application, Security, System**, and **Sysmon** events. 

- The default `inputs.conf` file is found in: `C:\Program Files\SplunkUniversalForwarder\etc\system\default\inputs.conf`.
- But **never** edit files in the `default` directory, as these are overwritten with updates. Instead, we’ll create a new `inputs.conf` file in the `local` directory.
- **Open Notepad as Administrator** (right-click Notepad, select **Run as Administrator**, then **Yes** when prompted).
- Copy the following configuration and paste it into Notepad:

    ```plaintext
    [WinEventLog://Application]
    index = endpoint
    disabled = false

    [WinEventLog://Security]
    index = endpoint
    disabled = false

    [WinEventLog://System]
    index = endpoint
    disabled = false

    [WinEventLog://Microsoft-Windows-Sysmon/Operational]
    index = endpoint
    disabled = false
    renderXml = true
    source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
    ```

- **Explanation**: This configuration tells the Splunk UF to capture logs from the Application, Security, and System events, as well as detailed Sysmon logs. All logs are directed to an **index named `endpoint`** on our Splunk server.
-  Save this file as `inputs.conf` in:
    ```
    C:\Program Files\SplunkUniversalForwarder\etc\system\local
    ```
- For the new configuration to take effect, you must restart the Splunk UF service:

1. Open **Services** (type “Services” in the Windows search bar and run as Administrator).
2. Scroll down and find **SplunkForwarder Service**.
3. **Double-click** to open it, go to the **Log On** tab.
   - Ensure it’s set to **Local System Account** rather than **NT Service** to avoid permissions issues when collecting logs.
4. **Click Apply** and then **OK**.
5. Stop and restart the SplunkForwarder Service to apply the `inputs.conf` updates.

Once restarted, the Splunk UF will start forwarding the selected logs to your Splunk server under the `endpoint` index. Just remember, if your Splunk server does not have an `endpoint` index, these logs won’t be ingested! You’ll need to create an `endpoint` index on the Splunk server to receive these events. Here’s how to set it up:

1. Open a browser and navigate to your Splunk instance (`http://<your-splunk-server-ip>:8000`). Log in with your Splunk credentials.
   <p align="center"><img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/splunk.PNG" width="70%"/> </p>

2. In the top-right corner, click on **Settings**. Under **Data**, select **Indexes**. This will take you to the Index Management page.

3. **Create a New Index**  
   - Click **New Index** at the top-right of the page.
   - In the **Index Name** field, type `endpoint`. This name must match exactly with what was configured in the `inputs.conf` on the Universal Forwarder.
   - **Choose Index Type**: Leave it as **Events**.
   
4. **Save the Index**  
   After entering the required information, click **Save**. The `endpoint` index is now created and ready to receive data.

5. From the Splunk Home screen, select **Search & Reporting**.

6. In the search bar, type the following command to verify that data is being ingested into your new index:
   ```plaintext
   index=endpoint
   ```
<p align="center">
  <img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/index.PNG" width="70%" />
</p>


Follow these same steps to install the Splunk Universal Forwarder on any other Windows machines in your environment. Each forwarder will report to your Splunk server, consolidating your logs and making monitoring easier.

# Brute Force Attack Using Crowbar in Kali Linux with Rockyou Wordlist

In this section, we’ll dive into performing a brute-force attack using **Crowbar**, one of the popular tools in **Kali Linux**. We'll be leveraging the famous **Rockyou wordlist** to guess passwords, but for simplicity, we’ll limit ourselves to the first 20 entries of this list and intentionally add our target domain user's password so we can quickly verify the success of the attack.

**Crowbar** is an easy-to-use tool designed to perform brute-force attacks against various services such as SSH, RDP, and HTTP. Today, we'll target RDP, but before proceeding, we need to ensure that the RDP service is enabled on the target machine.

### Step-by-Step Guide for Brute-Force Attack with Crowbar

#### Step 1: Install Crowbar (If Not Already Installed)
Crowbar is usually pre-installed in Kali Linux, but if you find that it's not installed, you can easily get it using the following commands:

```bash
sudo apt update
sudo apt install crowbar
```

#### Step 2: Prepare the Wordlist

Kali Linux includes the Rockyou wordlist by default, located at `/usr/share/wordlists/rockyou.txt.gz`. For our demonstration, we’ll extract just the first 20 passwords and add our domain user's password to the list. Here's how to do it:

1. First, extract the first 20 passwords from the Rockyou wordlist:
   ```bash
   head -n 20 /usr/share/wordlists/rockyou.txt > password.txt
   ```
2. Next, add our domain user's password to the wordlist to ensure we get a successful result:

   ```bash
   echo "yourpassword" >> password.txt
   ```
Note: Replace `yourpassword` with the actual password for the domain user you're targeting.

3. Launch the Brute-Force Attack

Now that our wordlist is ready, it’s time to launch the brute-force attack using **Crowbar**. In this example, we’ll target RDP, but Crowbar also works with other services like SSH and HTTP. To start the attack, run the following command:

```bash
crowbar -b rdp -u <target-user> -C password.txt -s <target_ip/32>
```
- `-b rdp`: Specifies that we want to brute-force a RDP service.
- `-u <target-user>`: Specifies the username of the target (the domain user we are attacking).
- `-C password.txt`: Points to the wordlist we’ve created.
- `-s <target_ip/32>`: Replace `<target_ip>` with the IP address of the target machine. `/32` means we are only target one machine.

<p align="center">
<img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/bruteforce.PNG" width="70%"/>
</p>

Now, we can head over to Splunk and search for logs of the brute-force attack. By looking at the timestamps we can tell several login attempts were made within a short period of time.

<p align="center">
  <img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/brute-force-log.png" width="70%"/>
</p>

# Invoke Atomic Red Team To Generate Logs

In this section, we'll explore how to use **Atomic Red Team** to simulate an attack and generate corresponding logs in **Splunk**. Let’s dive into the installation and execution process step-by-step.

#### Step 1: Open PowerShell as Administrator

Since we're operating within an Active Directory (AD) environment, we'll use administrator credentials to run PowerShell with administrative privileges.

#### Step 2: Set Execution Policy

1. To allow the execution of scripts necessary for installing Atomic Red Team, we need to adjust the PowerShell execution policy.
   ```powershell
   Set-ExecutionPolicy Bypass CurrentUser
   ```
2. * When prompted, type `Y` and press `Enter` to confirm.

#### Step 3: Add an Exclusion for the C:\\ Drive

Before installing Atomic Red Team, it's essential to prevent Microsoft Defender from flagging it as malicious by adding an exclusion for the entire `C:\` drive.

1. * Search for **Windows Security** and goto  **Virus & threat protection**.
2. * Click on **Manage settings** under the **Virus & threat protection settings** section.
3. * Scroll down to **Exclusions** and click **Add or remove exclusions**.
   * Click **Add an exclusion** and select **Folder**.
   * Browse to and select `C:\` to exclude the entire drive.
   
   > **Note:** Adding a full drive exclusion can pose security risks. Ensure this is done in a controlled lab environment and not on production systems.

#### Step 4: Install Atomic Red Team Framework

1. **Run the Installation Command:**
   ```powershell
   IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing);
   Install-AtomicRedTeam -getAtomics
   ```
2. When prompted, type `Y` and press `Enter` to install the necessary dependencies.

### Step 5: Navigate to Atomic Red Team Directory
Go to the directory `C:\AtomicRedTeam\atomics`. Here, you’ll find technique IDs that map back to the MITRE ATT&CK framework.

Visit the MITRE ATT&CK website to select the technique you want to simulate. For this example, we'll create a local account under the Persistence tactic.
* **Navigate to MITRE ATT&CK:**
  * Go to **Persistence** > **Create Account**.

* **Select Technique ID:**
  * Choose **Local Account** with Technique ID `T1136.001`.
  * Ensure this ID exists in your `C:\AtomicRedTeam\atomics` directory.
With everything set up, it’s time to invoke the atomic test to generate logs.

1. **Run the Atomic Test:**

   ```powershell
   Invoke-AtomicTest T1136.001
   ```
2. This command will create telemetry for local account creation. Take a look at the username,it could be `NewLocalUser`
3. Now, head over to your Splunk instance to search the logs.
4. In the Search & Reporting app, enter the following search query:
   ```plaintext
   index=endpoint NewLocalUser
   ```
5. You should see an entry for `NewLocalUser`, indicating that the local account creation was successfully logged.(Note: you may have to wait a little bit to see the logs)

<p align="center">
<img src="https://github.com/Md-Jamiul-Haque/Splunk-Projects/blob/main/mitre-attack.png" width="80%"/>
</p>

# Conclusion
<div align="justify">
Setting up a SIEM lab environment using Splunk, Sysmon, and Atomic Red Team provides a powerful way to monitor, detect, and analyze potential security incidents in real-time. By configuring Splunk Universal Forwarder to capture critical logs from Windows hosts and setting up a static IP for Splunk on Ubuntu, we created a strong foundation for centralized log management. The simulated brute-force attack using Kali Linux and the local account creation with Atomic Red Team allowed us to test our configurations and observe how Splunk detects these events, making it an effective training tool.
Whether you’re exploring cybersecurity for the first time or fine-tuning your skills, this setup offers valuable hands-on experience with log analysis and detection techniques. With this foundation in place, you can now dive deeper into creating custom detection rules, analyzing trends, and enhancing your Splunk queries to broaden your expertise in threat monitoring and response.
</div>

