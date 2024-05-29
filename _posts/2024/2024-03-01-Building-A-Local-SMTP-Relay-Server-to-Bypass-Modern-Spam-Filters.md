---
title: Building A Local SMTP Relay Server to Bypass Modern Spam Filters
description:
author: BW
date: 2024-03-01 20:00:00 - 0500
categories: [CyberSecurity, Phishing]
tags: [phish]     # TAG names should always be lowercase
---


Setting up a local SMTP server on a VPS from scratch to implement TLS, SPF, DKIM, DMARC and other modern checks to improve overall mail deliverability and bypass modern spam filters like Gmail, yahoo, outlook etc. Building a local SMTP also allows to send/receive unlimited mails for free as long as the hosting provider permits it (outbound 25,587) and you haven’t run out of internet bandwidth.

The default ports used by SMTP are 25, 465 (Exchange) and 587 that are meant to be used for submissions from your e-mail client to the e-mail server and higher ports are used for relaying between SMTP-server.

Note: This won't work in Azure as they block port 25 due to the prevalence of abuse, I'm using hostwinds.com in this case.

Mail User Agent (MUA): This is a (part of a) program connecting to a SMTP-server in order to send an email. Most likely this is your Outlook, Thunderbird, whatever.

Mail Transfer Agent (MTA): The transport service part of a program. They receive and transfer the emails. This might be an Exchange server, an internet facing gateway and so on.

A workflow of an email´s travel from one user to another would look like so: 
__MUA → MSA → MTA → internet → MTA → MDA → MUA__

A “relay” SMTP system receives mail from an SMTP client and transmits it, without modification to the message data other than adding trace information, to another SMTP server for further relaying or for delivery.

Setting up a Message Transport System (MTS) aka SMTP server (Postfix)

Postfix is a light, easy to use MTS which serves 2 primary purposes:

Transporting email messages from a mail client/mail user agent (MUA) to a remote SMTP server.
Accepts emails from other SMTP servers.

We will configure postfix for a single domain in this tutorial. Before we install postfix note to do the following before.

Set Hostname and DNS records

Postfix uses the server’s hostname to identify itself when communicating with other MTAs. A hostname could be a single word or a FQDN.

Note: We will use example.com as our registered domain as an example here.

Make sure your hostnames set to a FQDN such as mail.example.com by using the command: sudo hostnamectl set-hostname mail.example.com

Gracefully reboot your server using init 6 after.

Set up DNS records

MX records tell other MTA’s that your mail server mail.example.com is responsible for email delivery for your domain name.


```
MX record    @       	mail.example.com

An A record maps your FQDN to your IP address.

mail.example.com    	<ip-addr>
```


Installing Postfix

Run this in a terminal:
```shell
sudo apt update
sudo apt install postfix -y
```
While installation you will be asked to select a type for mail configuration. Select Internet Site. This option allows Postfix to send emails to other MTAs and receive emails from other MTAs.



Next enter your domain name when prompted for the system mail name as your domain name without “mail” that is just example.com.

This ensures that your mail address naming convention would be in the form of -

[-] name@example.com and not,

[x] name@mail.example.com.

Use a valid subdomain replacement if you would need to implement one, it will work.



Once installation is complete a /etc/postfix/main.cf config file would be automatically generated along with postfix starting up.

Check your current Postfix version using:
```shell
postconf mail_version.
```
Use Socket Statistics - ss utility to check if postfix is running on port 25 succesfully:
```shell
sudo ss -lnpt | grep master
```
If you’d like to view the various binaries shipped along with postfix check them out with:
```shell
dpkg -L postfix | grep /usr/sbin/
```
Sendmail is a binary place at /usr/sbin/sendmail which is compatible with postfix.

Send out your first testmail to your test email account using :
```shell
echo "test email" | sendmail your-test-account@gmail.com
```
Or you could install mailutils using sudo apt install mailutils . Just type mail and follow along the prompts entering the required fields and hitting Ctrl+D once done to send the mail.

Note: The email might land through into your primary right away but could be potentially flagged by other stronger MTA’s and their spam filters.

In case your hosting provider has blocked outbound port 25, verify it using:
```shell
telnet gmail-SMTP-in.l.google.com 25
```
If you see a status showing "Connected" --> outbound 25 works successfully. Use quit to quit the command.

Head on over to your Gmail inbox and open up the mail. Click on the drop down below the Printer icon to the right as shown in the screenshot –> next click on show original. –> next click on the Copy to clipboard button to copy all contents.



For DKIM signing we will use OpenDKIM

Step 1. Installing OpenDKIM on Your Postfix Server

Start with installation:
```shell

sudo apt install opendkim opendkim-tools

sudo systemctl start opendkim

sudo systemctl enable opendkim
```
Create the dkim keys
```
opendkim-genkey -b 2048 -D /etc/opendkim/ --domain example.com –selector mail
```
Creates two files mail.private and mail.txt, we’re going to copy the contents of mail.txt to add to our dns records
```
sudo chown -R opendkim:opendkim /etc/opendkim
```
Step 2: Configure OpenDKIM

Edit OpenDKIM main configuration file
```shell
sudo vi /etc/opendkim.conf
```
Add:
```
Autorestart                     	yes

AutorestartRate            	10/1h

Uncomment #LogWhy and change from no to yes
```
Find the “Mode v” line, and change it to “Mode sv”. By default, OpenDKIM is set to verification mode (v), which verifies the DKIM signatures of receiving email messages. Changing the mode to “sv,” will let us activate the signing mode for outgoing emails.

Change “Mode v” to “Mode sv”

Comment out
```
#Socket	local:/run/opendkim/opendkim.sock
```
and Uncomment
```
#Socket	inet:8891@localhost
```
In the same OpenDKIM Configuration file, add these lines to the bottom of the file:
```sass
ExternalIgnoreList refile:/etc/opendkim/TrustedHosts

InternalHosts refile:/etc/opendkim/TrustedHosts

KeyTable refile:/etc/opendkim/KeyTable

SigningTable refile:/etc/opendkim/SigningTable

SignatureAlgorithm     	rsa-sha256
```
{: file='/etc/opendkim.conf'}

Now create the files TrustedHosts, KeyTable, and SigningTable with touch

In the TrustedHosts file we’ll add:
```sass
127.0.0.1

::1

Public ip of server or any other server you plan to use with it, can use curl ip.me to find that out

Example.com

*.example.com
```
{: file='/etc/opendkim/TrustedHosts'}

KeyTable file:
```sass
mail._domainkey.example.com example.com:mail:/etc/opendkim/mail.private
```
{: file='/etc/opendkim/KeyTable'}
SigningTable file:
```sass
*.example.com mail._domainkey.example.com
```
{: file='/etc/opendkim/SigningTable'}
```shell
Sudo systemctl restart postfix

Sudo systemctl restart opendkim
```


You can test the email sending with the below command like so:
```
/usr/sbin/sendmail john.doe@gmail.com

From: mike@example.com

Subject: Test Subject

Test body

CTRL-D to close and send
```
Congratulations! Now you can use this with gophish and start your phishing campaign!
