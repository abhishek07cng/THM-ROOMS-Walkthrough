# TryHackMe Room Writeup — Pickle Rick

**Platform:** TryHackMe  
**Room Type:** Beginner  
**Focus Areas:** Reconnaissance, Web Enumeration, Command Execution, Privilege Escalation  

---

## Overview

This room was completed as part of my cybersecurity practice to improve my skills in:

- Network scanning
- Web application enumeration
- Identifying hidden resources
- Command execution analysis
- Basic privilege escalation techniques

The goal of this write-up is to document the methodology and learning outcomes rather than provide a step-by-step solution.

---

## 1. Reconnaissance

I began by performing network scanning to identify open ports and running services.

### Tools Used:
- Nmap

The scan revealed:
- An SSH service
- An HTTP web server

Since web services are often a common attack surface, I prioritized further investigation of the HTTP service.

### Skills Practiced:
- Service version detection
- Identifying potential attack vectors
- Initial attack surface analysis

---

## 2. Web Enumeration

After accessing the website, I inspected:

- Page source code
- Comments
- Hidden information

This helped identify potential usernames and clues embedded in the application.

### Skills Practiced:
- Manual inspection of web applications
- Identifying hidden information in source code
- Understanding basic information disclosure vulnerabilities

---

## 3. Directory Enumeration

I performed directory brute-forcing to discover hidden paths and files.

### Tools Used:
- Gobuster
- FFUF
- Dirsearch
- Nikto

This process revealed several interesting endpoints, including:

- Administrative pages
- Configuration files
- Standard hidden directories

### Skills Practiced:
- Directory brute-forcing
- Using wordlists effectively
- Understanding web application structure
- Identifying exposed resources

---

## 4. robots.txt Analysis

I checked the `robots.txt` file to identify any disallowed or hidden paths.

This is a common reconnaissance technique that can reveal valuable information.

### Skills Practiced:
- Understanding search engine exclusion files
- Identifying potential hidden content

---

## 5. Authentication Testing

A login portal was discovered during enumeration.

Using previously identified information, I attempted authentication and gained access to the application.

After logging in, I found a command execution interface.

### Skills Practiced:
- Understanding authentication mechanisms
- Testing for credential reuse
- Recognizing weak access control

---

## 6. Command Execution Analysis

The application allowed system command execution.

I tested basic commands to:

- List files
- Explore directories
- Identify restricted areas

This demonstrated a **command injection / insecure command execution vulnerability**.

### Skills Practiced:
- Identifying insecure input handling
- Exploring file systems via command interface
- Understanding server-side execution risks

---

## 7. File System Exploration

Using the command interface, I explored:

- User directories
- System directories
- Restricted areas

This helped identify important files required to complete the room objectives.

### Skills Practiced:
- Linux file system navigation
- Understanding permission structures
- Reading files with appropriate access

---

## 8. Privilege Escalation

To access restricted files, I explored permission configurations and evaluated available system commands.

This involved analyzing:

- User privileges
- Sudo permissions
- Root directory access

Through proper privilege analysis, I was able to retrieve the final required information.

### Skills Practiced:
- Basic privilege escalation concepts
- Checking sudo permissions
- Understanding Linux security boundaries

---

## Key Learning Outcomes

From this room, I strengthened my understanding of:

- Network reconnaissance
- Web application enumeration
- Information disclosure
- Command injection vulnerabilities
- Linux file system structure
- Basic privilege escalation techniques

---

## Tools Used

- Nmap
- Gobuster
- FFUF
- Dirsearch
- Nikto
- Linux command-line utilities

---

## Conclusion

This room helped reinforce the importance of systematic enumeration and structured thinking during penetration testing.

Key takeaway:

> Always enumerate thoroughly before attempting exploitation.

Proper reconnaissance significantly increases the chances of identifying vulnerabilities efficiently.

---

**End of Write-Up**