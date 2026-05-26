# In-memory-Mimikatz-execution-and-powershell-obfuscation-detection-using-Sysmon

---

# Objective

The objective of this project is to simulate a stealthy in-memory cyberattack using Mimikatz and obfuscated PowerShell commands, then detect the malicious behavior using Sysmon and Splunk.

---

# Tools Used

- Sysmon
- Splunk
- Mimikatz
- PowerShell

---

# Task

1. Execute Mimikatz using encoded PowerShell in a fileless (in-memory) manner.  
2. Detect PowerShell execution with encoded/Base64 commands using Sysmon Event ID 1.  
3. Monitor LSASS process access attempts using Sysmon Event ID 10.  
4. Detect process injection or remote thread creation using Sysmon Event ID 8.  
5. Correlate encoded PowerShell execution followed by LSASS access within a short time window to confirm malicious activity.



# Mimikatz Overview

## 1. What is Mimikatz

Mimikatz is an open-source tool created by Benjamin Delpy, originally developed to demonstrate security vulnerabilities in the way Windows handles authentication. It allows for the extraction of plaintext passwords, hashes, Kerberos tickets, and other sensitive information from memory.

It can be used by attackers to perform malicious activities such as stealing credentials and lateral movement. Also, it can be used by security defenders and penetration testers to identify vulnerabilities, test system detection, and improve system defenses.

---

## 2. Mimikatz Features

The key features of Mimikatz include:

- Credential extraction: Recovering usernames and passwords stored in memory.
- Ticket manipulation: Modifying and injecting Kerberos tickets for authentication.
- Credential dumping: Extracting stored user credentials from the Security Accounts Manager (SAM) database and Active Directory.
- Kerberos attacks such as Golden Ticket and Over-Pass-the-Hash.
- NTLM attacks like Pass-the-Hash.
- Defense evasion by clearing Windows event logs or injecting into legitimate processes.

---

## 3. How Mimikatz Works

Mimikatz takes advantage of Windows authentication techniques and APIs, such as Local Security Authority (LSA) and Security Support Provider Interface (SSPI). This enables it to manipulate authentication tickets and extract user credentials.

---

# Introduction

This report represents a complete purple team scenario, in which an attacker simulates an attack by executing the Mimikatz tool fully in memory to evade detection and remain stealthy. On the defensive side, Sysmon is used to monitor system activity and detect suspicious processes attempting to access LSASS (Local Security Authority Subsystem Service).

LSASS access enables attackers to extract user credentials from memory, which can then be used to perform post-exploitation activities such as gaining privileged access, lateral movement, and unauthorized access to sensitive resources.

---


# 1. Attack Simulation Using Mimikatz Tool

## 1.1 Attack Simulation

To simulate the attack, the following steps were performed:

### 1. Disable Windows Security

To allow the execution of the attack, Microsoft Defender Antivirus was temporarily disabled. This step ensures that security controls do not block the execution.

### 2. Prepare the PowerShell command and store it in a variable

`Invoke-Mimikatz.ps1` is a special PowerShell script allowing PowerShell to perform remote fileless execution of this threat.

Fileless execution enables loading a binary into process memory without touching the hard disk. When a fileless binary is loaded directly into memory, it remains invisible to file-scanning antivirus solutions.

### 3. Convert the command into binary

### 4. Encode the command using `ToBase64String`

### 5. Run PowerShell with encoded command

<img width="945" height="738" alt="image" src="https://github.com/user-attachments/assets/fa3317b1-f427-4c4f-978d-f71466512652" />

---

# 1.2 Observed Suspicious Behavior

During the simulation, several abnormal behaviors were observed that indicate potential malicious activity:

## PowerShell with encoded command

The presence of PowerShell encoded commands is a strong indicator of suspicious activity commonly used to evade antivirus and Defender detection.

## High entropy command line

The command line encoded with Base64 is characterized by a long random string composed of upper/lower case characters and numbers. This is a suspicious technique used by attackers to obfuscate payloads and commands.

## LSASS.exe access attempts

Another suspicious behavior observed is the attempt to access `lsass.exe`, which stores sensitive information in memory such as usernames, domains, and NTLM hashes.

By accessing `lsass.exe`, attackers can extract user credentials from memory, enabling further malicious activities such as privilege escalation or lateral movement.

---

# 1.3 Result of Attack

The simulation attack resulted in the extraction of user credentials from memory. This behavior is malicious and indicates a potentially suspicious process attempting to compromise the device.

---

# 2. Detection of the Attack Using Sysmon

To detect suspicious activity, Sysmon was configured to detect:

- New process execution
- Process access to `lsass.exe`
- Remote thread creation

---

# 2.1 What is Sysmon

Sysmon (System Monitor) is a Windows system service developed by Microsoft as part of the Sysinternals Suite (a collection of advanced system utilities). It tracks processes, network connections, registry changes, and driver loading inside Windows.

---

# 2.2 Monitor Event ID 1 – Process Creation

I configured Sysmon to alert on any PowerShell command executed with `-EncodedCommand`.

<img width="944" height="398" alt="image" src="https://github.com/user-attachments/assets/cbfab533-e2ae-43b5-846f-1087e876c4f6" />


Then, I updated the Sysmon configuration by adding the created rule.

Next, I ran a PowerShell command with `-EncodedCommand`.

<img width="983" height="787" alt="image" src="https://github.com/user-attachments/assets/05c38fa6-e449-4689-9efc-1c53b2560d15" />

> Sysmon successfully generated Event ID 1 after executing a PowerShell command with `-EncodedCommand`.

<img width="944" height="79" alt="image" src="https://github.com/user-attachments/assets/21fdd47a-fe21-4dc7-b600-7f3e860cdb77" />

---

# 2.3 Monitor Event ID 10 – Process Access

I configured Sysmon to alert on any process accessing `lsass.exe` with high access rights and memory read privileges.

<img width="982" height="392" alt="image" src="https://github.com/user-attachments/assets/542346ea-d16e-4293-b544-638add4a69dc" />


Then, I updated the Sysmon configuration by adding the Process Access rule.

<img width="945" height="229" alt="image" src="https://github.com/user-attachments/assets/c9af771e-b50f-4083-981f-f50b748cb23a" />

Next, I executed the required command.

<img width="944" height="229" alt="image" src="https://github.com/user-attachments/assets/343cfa4f-855d-4b43-98c3-0f3bf17133d4" />

> Sysmon successfully monitored `lsass.exe` process access and generated Event ID 10.

<img width="903" height="779" alt="image" src="https://github.com/user-attachments/assets/b78ca1b3-a06d-4df0-b537-5f60e996838e" />

---

# 2.2.3 Monitor Event ID 8 – RemoteThreadCreation

The CreateRemoteThread event detects when a process creates a thread in another process.

In Pass-the-Hash attacks, after gaining access to the compromised machine, attackers perform post-exploitation activities to extract user credentials stored in memory by accessing `lsass.exe`. Then, they impersonate the system using NTLM hashes to authenticate without needing passwords to perform lateral movement, credential theft, or gain privileged access.

To create a remote thread, I used user credentials gathered after accessing lsass.exe:

<img width="945" height="640" alt="image" src="https://github.com/user-attachments/assets/c75d1c08-2e2f-48b7-816a-dbe44b5c0c98" />


Then, the command was injected into the process executing Mimikatz.

<img width="944" height="412" alt="image" src="https://github.com/user-attachments/assets/6710a861-bc9b-41a4-924b-0ee81dc72e58" />


Next, a Sysmon rule was created to detect new remote thread creation.

<img width="944" height="398" alt="image" src="https://github.com/user-attachments/assets/b223793d-6e46-4957-be47-06b0bf95c2cc" />

Then, the Sysmon configuration was updated by adding the CreateRemoteThread rule.

<img width="945" height="202" alt="image" src="https://github.com/user-attachments/assets/9ba76c4b-5952-4ef2-83d1-0c8bbee9a12f" />

> Sysmon successfully detected and alerted on remote thread creation.

<img width="983" height="743" alt="image" src="https://github.com/user-attachments/assets/b0c8ce5e-5677-45fc-81c9-1c91e86d6875" />

---

# 3. Detection Based Correlation Using Sysmon and Splunk

## 3.1 Splunk Configuration

<img width="945" height="420" alt="image" src="https://github.com/user-attachments/assets/11496d16-8b91-454e-be50-16921262f2f2" />

After installing Splunk, it was configured to receive data logs from the Windows operating system.

<img width="983" height="629" alt="image" src="https://github.com/user-attachments/assets/7ca1f67e-cfe6-4740-a9b3-4a8ae8362a8a" />

<img width="983" height="552" alt="image" src="https://github.com/user-attachments/assets/f5ce5672-0305-41eb-a22e-26ac77c355d7" />

<img width="983" height="510" alt="image" src="https://github.com/user-attachments/assets/db19c41f-877a-4fe3-a05c-3fed43b79c66" />

<img width="983" height="517" alt="image" src="https://github.com/user-attachments/assets/4b446092-0a75-4922-90dd-ebc3fe9c5a7e" />

<img width="983" height="616" alt="image" src="https://github.com/user-attachments/assets/5f2a2c17-db3d-43d1-a06d-89e1571335ec" />


Since no Sysmon logs were present in Splunk, Splunk Universal Forwarder was installed on the Windows machine to collect, forward, and monitor Sysmon logs.

### Steps performed

1. Enable receiving logs on Splunk by configuring listening on port `9997`.

<img width="983" height="516" alt="image" src="https://github.com/user-attachments/assets/fc3cee35-ec45-4a3d-85cd-248f153a33d9" />

<img width="983" height="316" alt="image" src="https://github.com/user-attachments/assets/4ce13382-435e-4822-ab67-705979affa0f" />

<img width="983" height="313" alt="image" src="https://github.com/user-attachments/assets/ea714f7c-cf9a-4748-900d-71ea063de8de" />

2. Create the Sysmon index.

<img width="983" height="462" alt="image" src="https://github.com/user-attachments/assets/fdce8cd9-b7df-437c-b86d-17f693ee4c6b" />

3. Configure Splunk Universal Forwarder to collect Sysmon logs.

In the windows machine, I add inputs.conf file in the folder C:\Program Files\SplunkUniversalForwarder\etc\system\local\ to enable Splunk forwarder to collect logs from Sysmon.

<img width="982" height="439" alt="image" src="https://github.com/user-attachments/assets/23121db7-1306-4c98-b427-d980f1dd6b05" />

4. Restart Splunk.

<img width="945" height="495" alt="image" src="https://github.com/user-attachments/assets/068a0b7f-2955-4d38-b690-6e1bc5b766ad" />

After restarting Splunk, no sysmon were present.

<img width="983" height="315" alt="image" src="https://github.com/user-attachments/assets/3bf63774-519b-4a85-8bdc-3175c302d2d5" />

Splunk Forwarder runs as a network service, which may not have permission to read Sysmon logs. It was configured to run as Local System to obtain the required permissions to access Sysmon event logs.

<img width="983" height="632" alt="image" src="https://github.com/user-attachments/assets/f1c51b65-caa4-4d7e-83d9-11baead741c1" />

To enable Splunk forwarder to run as local system I executed the following command:

<img width="983" height="526" alt="image" src="https://github.com/user-attachments/assets/63ef6bf3-b6c2-40af-9403-e7e82ce41a29" />


Finally, Splunk successfully monitored Sysmon logs.

<img width="983" height="540" alt="image" src="https://github.com/user-attachments/assets/892a5c52-a579-49ce-b3bc-bc884d4e0389" />

---

# 3.1 Detection Based Correlation

The following Splunk SPL was used to alert if:

1. Encoded PowerShell execution occurred
2. Followed by LSASS access
3. Within a short time window
4. From user context

<img width="945" height="466" alt="image" src="https://github.com/user-attachments/assets/ed3fee4d-d261-4050-9090-0dad5e6c1bc8" />

<img width="983" height="505" alt="image" src="https://github.com/user-attachments/assets/1d8c2462-72ff-4fad-beec-e109214d39fe" />



---

# Conclusion

By integrating Sysmon with Splunk, it was effectively possible to monitor system activities. The Sysmon configuration allowed detection of newly created malicious processes, any access to `lsass.exe`, and remote thread creation especially for Sysmon.

Mimikatz is a powerful hacking tool that enables attackers to steal credentials such as hashes, plaintext passwords, and Kerberos tickets that can later be used for post-exploitation.

To detect and prevent Mimikatz execution:

- Regularly update and patch operating systems.
- Use strong and unique passwords for all accounts.
- Enforce password policies requiring regular password changes.
- Enable Credential Guard in Windows 10 and Windows Server 2016 to help protect credentials from being extracted by tools like Mimikatz.
- Implement Two-Factor Authentication (2FA) for sensitive systems and privileged accounts.
- Apply the principle of least privilege by limiting user privileges and restricting administrative accounts to the minimum required access.
- Deploy Endpoint Detection and Response (EDR) solutions capable of detecting and blocking known Mimikatz signatures and behaviors.
- Regularly update EDR solutions to ensure the latest threat intelligence is used.
- Implement a Security Information and Event Management (SIEM) solution to automatically monitor security logs for suspicious activities such as failed logins and unusual system access.
