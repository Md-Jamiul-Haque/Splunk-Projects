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
