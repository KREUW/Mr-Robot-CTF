# Walkthrough: Mr. Robot Vulnerable Machine

## Introduction
This walkthrough documents the process of penetrating the Mr. Robot vulnerable machine. The objective was to exploit the machine to uncover three hidden keys, simulating an attack on a fictional system inspired by the "Mr. Robot" TV series.

## Tools Used
- **Nmap**: For scanning the target's open ports and running services.
- **Gobuster**: For directory enumeration.
- **Burp Suite Community Edition**: For capturing and analyzing web traffic.
- **Hydra**: For brute-forcing passwords.
- **John the Ripper**: For cracking hashed passwords.
- **Netcat (nc)**: For setting up backdoor listening.
- **PHP Reverse Shell**: Leveraged from PentestMonkeys for creating a reverse shell.

### Discovery and Enumeration
- **Initial Scanning**:
  ```bash
  nmap -sV -sC -oA scans/intial 10.10.37.51
  ```
  ![Robot scan 1](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/6b3b0876-d983-4b09-adea-42940ffd0ac6)

  - This scan revealed several important details:
- Port 80/tcp: Open, running Apache httpd. This indicates a web server is active.
- Port 443/tcp: Open, running Apache httpd over SSL, suggesting secured web traffic.
- Port 22/tcp: Closed, which is typically used for SSH (Secure Shell).
- Most importantly it notes SMB vulnerabilty MS17-010


### **Web Enumeration**:
``` gobuster dir -u http://10.10.37.51 -w /usr/share/dirb/wordlist/small.txt -t 100 -q -o scan/gobuster-small.txt ```
  Initial inspection of the website from the browser and via command line tools highlighted several key discoveries:

  ![gobuster 1](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/a857a24e-1deb-4d97-b0c5-77268e073006)


## This process identified several directories and files, including:
- /wp-login.php: A WordPress login page, which is often a target for brute-force or credential stuffing attacks.
- /fsocity.dic: A downloadable file that turned out to be a wordlist, likely useful for further password attacks.


## Web Server Investigation:
- A manual browse to the IP address over HTTP (port 80) and HTTPS (port 443) was performed to understand the web content and server configuration.
- The following was on the landing page:

![Screenshot 2024-04-25 063634](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/0c29509b-1e89-4c1c-a279-95219d59dadb)

- The Apache server headers were visible, but no significant information (like server version) was leaked, reducing some avenues for direct server exploitation.
robots.txt Review:
- Robots.txt files are used by websites to communicate with web crawlers about what areas of the site should not be processed or scanned.
- Upon inspection, the robots.txt on the target revealed a disallowed entry:

![robots reveal](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/c2fa6fbe-b3c2-4962-80c3-22b81170b9b7)
- Inspection of the page element also reveals this mesasge:
![Screenshot 2024-04-25 063754](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/f7606c47-5b55-41d7-a4b4-3d9bed358ccc)

### WordPress Login Page Exploitation:

The discovery of a WordPress login page (wp-login.php) marked a significant potential entry point. WordPress sites can be vulnerable to brute-force attacks if not properly secured.

![WP login](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/ab169b0e-c031-4038-aa98-35882602f579)

### Traffic Analysis with Burp Suite:

Initial attempts to log in through the WordPress admin panel were captured using Burp Suite. This helped understand how login requests were made and how the server responded to failed attempts.
It was noted that the server provided specific error messages for "invalid username" and "incorrect password," which could be exploited to confirm valid usernames before attempting to brute-force passwords.

![error handlign 1](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/583be6c3-f8af-4720-a570-718c07aa6591)

![WP login capture](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/5e48c925-43f3-4c74-9127-a78991b37f38)

### Username Enumeration with Hydra:
  - **Brute Forcing with Hydra**:
  - 
``` hydra -L fsocity.dic -p test 10.10.37.51 http-post-form "/wp-login.php:log=^USER^&pwd=^PWD^:Invalid username" -t 30 ```

This process iteratively tested usernames from the **fsocity.dic** wordlist with a placeholder password. The response "invalid username" indicated a failed attempt, whereas a different error (or no error related to the username) suggested a potentially valid username. This identified **Elliot** as a valid username.

## Password Brute-forcing for **Elliot** :

With a valid username confirmed, the next step was to brute-force the password for the **Elliot** account:

``` hydra -l Elliot -P fsocity.dic 10.10.37.51 http-post-form "/wp-login.php:log=^USER^&pwd=^PWD^:The password you entered for username" -t 30 ```
    
 - The password was discovered to be **ER28-0652** enabling full access to the WordPress admin dashboard.
![login ](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/28e7ba72-a01c-4b00-9535-16d6c99b3739)

## Establishing a Reverse Shell:
- Using the administrative access gained on the WordPress site, a PHP reverse shell script was uploaded. This script, when accessed via a web browser, initiates a connection back to a listener set up by the attacker.
- The PHP reverse shell was configured to connect back to the attacker's machine at a specified IP and port.
- Before triggering the reverse shell, a listener was set up on the attacker's machine to receive the connection:
``` sudo nc -lvnp 53 ```
![nc caught sheel](https://github.com/KREUW/Mr-Robot-CTF/assets/151568256/bf4d0433-1a9c-41c9-9d1a-b96809b1e657)
- This command tells Netcat to listen verbosely (-v) on port 53 (-p 53), without using DNS (-n), and to keep the connection open for multiple back-and-forth transmissions (-p).
## Execution and Initial Shell Access:
Once the listener was ready, the uploaded reverse shell script was executed by simply accessing the file via the web browser. This triggered the shell to open a connection back to the listening Netcat session, providing shell access as the 'daemon' user.
## Navigating the System and Locating Sensitive Files:
In the shell, navigation to the user 'robot's home directory (/home/robot) revealed two files: key-2-of-3.txt and password.raw-md5. The permissions on key-2-of-3.txt prevented direct reading by the 'daemon' user.
## Cracking the Hash:
The password.raw-md5 file contained an MD5 hash of the robot's password. Using John the Ripper, this hash was cracked to reveal the password abcdefghijklmnopqrstuvwxyz.
## Switching to the Robot User:
With the cracked password, commands were issued to switch users to 'robot. Successful login as 'robot' granted access to read key-2-of-3.txt, capturing the second key.
## Exploring Further Privilege Escalation Vectors:
As 'robot', further system exploration and vulnerability scanning revealed that certain binaries had the SUID bit set, including an outdated version of Nmap.
## Using Nmap for Root Access:
The SUID bit set on Nmap allowed executing it with root privileges. Using Nmap’s interactive mode provided a method to spawn a root shell:
``` /usr/local/bin/nmap --interactive ```
<br>

``` !sh ``` 
- This sequence escalated privileges to root, allowing full control over the system and access to the third and final key stored in /root.
onclusion and Recommendations
Summary of Achievements:
In this comprehensive penetration test of the Mr. Robot vulnerable machine, we successfully navigated through multiple phases of a typical cyber attack:

Discovery and Enumeration: Identified open ports and potential entry points using Nmap and manual inspection.
Gaining Access: Exploited a vulnerable WordPress login mechanism to gain administrative access via brute-forcing with Hydra.
Privilege Escalation: Leveraged a PHP reverse shell and exploited a misconfigured Nmap binary to gain root access, allowing us to capture all three hidden keys.
These steps not only highlighted specific vulnerabilities but also demonstrated how layered security flaws can be exploited in succession to escalate privileges and breach systems.

# Lessons Learned:
The Mr. Robot VM challenge reinforced several critical security principles and exposed common vulnerabilities:

**Web Application Security:** The importance of securing web applications against brute force attacks and ensuring error messages do not disclose sensitive information.
**Proper Configuration:** The need for correct file permissions and system settings to prevent unauthorized access.
**System Updates:** Ensuring all applications, especially common tools like Nmap, are regularly updated to prevent exploitation of known vulnerabilities.

# Recommendations for Securing Systems:

**Regular Updates:** Keep all system software and applications updated to protect against known vulnerabilities.
**Strong Password Policies:** Implement and enforce strong password policies to guard against brute force attacks.
**Security Awareness Training:** Educate users and administrators about security best practices and the importance of non-disclosure of error details.
**Regular Security Audits:** Conduct frequent and thorough security audits to identify and rectify misconfigurations and vulnerabilities.
**Restrict SUID Binaries:** Limit the use of SUID binaries to those that are absolutely necessary, and regularly audit their necessity and security.

# Closing Thoughts:
This exercise underscores the importance of a multi-layered approach to security, combining technological solutions with user education and regular administrative review. By understanding and implementing these strategies, organizations can significantly enhance their resilience against cyber threats.
# Credits 
- Nmap: [https://nmap.org](https://nmap.org)
- GTFObins (for SUID exploits): [https://gtfobins.github.io](https://gtfobins.github.io)
- PentestMonkeys (for PHP Reverse Shell): [https://github.com/pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)
