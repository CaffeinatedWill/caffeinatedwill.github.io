---
layout: post
title: Setup a Free 2FA Solution on OWA
subtitle: ''
date: 2018-10-24T04:00:00.000+00:00
pageImageCard: "/assets/img/posts/03/compressedExchange.png"
content_img_path: ''
description: Long post on how to set up ADFS with privacyIDEA for a free MFA solution
  on Exchange
canonical_url: ''
tags:
- MFA
- Free MFA
- Free 2FA
- Free Two-Factor
- Free Multi-Factor
- privacyIDEA
- privacyIDEA Exchange
- Exchange MFA
- Architecture
published: true
quickbreach: true
redirect_from: 
 - /blog/setup-a-free-2fa-solution-on-owa/

---
In this post I will walk through how I setup the open-source MFA solution [privacyIDEA](https://github.com/privacyIDEA/privacyIDEA) to protect the OWA and ECP endpoints on my Exchange server. This is the setup that was shown in the demo from my Defcon26 talk "[One-Click to OWA](https://blog.quickbreach.io/blog/one-click-to-owa/)"

***

**Disclaimer:** I would NOT use this guide verbatim to stand up anything in production without performing more security and performance related background research, as well as applying other best practices (eg. use Exchange Edge Transport Servers, use a web proxy with AD FS, etc).

***

At the end of this guide you will have:

* A new open-source privacyIDEA MFA server to issue out and verify MFA tokens
* An Exchange CAS server that leverages Active Directory Federation Services(AD FS) to perform authentication on Outlook Web App (OWA) and Exchange Control Panel(ECP)
* A new AD FS server configured to require MFA for specific users/groups and process those MFA tokens through the privacyIDEA server

This guide is broken down into the following parts:

* **Part 1:** Create a PrivacyIDEA Server
* **Part 2:** Configure the PrivacyIDEA Server
* **Part 3:** Provide a User With a MFA Token
* **Part 4:** Setup an AD FS Server
* **Part 5:** Link AD FS to the PrivacyIDEA Server
* **Part 6:** Configure OWA & ECP to use AD FS for Authentication

Before we begin, let's assume we already have an Exchange server that went through a standard install with the Client Access Role(CAS) option selected. For the purposes of this blog post, we will be calling this Exchange CAS server _MAILMAN_.

***

## Part 1: Create a PrivacyIDEA Server

The first thing we need to do is stand up the privacyIDEA MFA server. This server will listen for requests passed from AD FS to check the validity of a given one-time password for a user, and respond appropriately.

1. Install and start up a standard Debian Stretch box. We will call this box _PISERVER_
2. Once you have the fresh install up, log into it and run the following commands to install the privacyIDEA server. Take note of the last line where you will need to swap in your own chosen username for the admin, and enter in the password for the account twice when prompted.

       # Install git/python
       apt update && apt install git python python-pip
       pip install virtualenv
       
       # Make a directory for the server & clone the data
       mkdir /privacyIDEA
       cd /privacyIDEA
       
       # This guide used the commit ff9427c80d4704bafff9fe4597f1ecffb230172d
       git clone https://github.com/privacyIDEA/privacyIDEA .
       
       # Generate an SSL cert for it to use (unless you have an internal CA to issue one)
       mkdir SSL
       openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout SSL/privacyIdeaCert.key -out SSL/privacyIdeaCert.crt
       
       # Install server requirements
       virtualenv venv
       source venv/bin/activate
       pip install -r requirements.txt
       
       # Set up server
       git submodule init
       git submodule update
       git pull --recurse-submodules
       ./pi-manage createdb
       ./pi-manage create_enckey
       
       # Set up admin on privacyIDEA server - you will be promted for your password, and to confirm it
       ./pi-manage admin add <InsertUsernameHere>

   Now that the server is installed, let's run it.

       ./pi-manage runserver -h 0.0.0.0 -p 443 --ssl-crt SSL/privacyIdeaCert.crt --ssl-key SSL/privacyIdeaCert.key

   You should now be able to open a browser to the server at https://_PISERVER_

***

## Part 2: Configure the PrivacyIDEA Server

Now let's log into the privacyIDEA server, hook it up to LDAP on our DC, and set up an MFA token for a of user.

1. Open a browser and navigate to https://_PISERVER_
2. Log in with the username and password you set up in the previous part
3. Click the "Config" tab on the top bar
4. From within the config tab, click on the "Users" tab as seen below
   ![](/assets/img/posts/03/ex1.png)
5. Now click "New ldapresolver" on the left sidebar.

   This form will create an ldapresolver which tells privacyIDEA how to access LDAP, and where in the LDAP structure to look for user objects. The privacyIDEA server requires a domain user account to query LDAP (unless anonymous LDAP binding is permitted - which is **BAD**), so be sure to create a basic user account for the privacyIDEA service. My user is called "privacyidea". Here are the basics on the ldapresolver configuration:

   | Field | Basic Description |
   | --- | --- |
   | Resolver Name | Some arbitrary name for the connector of your choosing |
   | Server URI | The LDAP URI to your domain controller (eg ldap://10.0.0.250) |
   | Base DN | The location within LDAP where the user objects are located |
   | Scope | How far down in LDAP to search - SUBTREE should be fine |
   | Bind DN | The domain\\username of the account to use for accessing LDAP |
   | Bind Password | Password for the Bind DN account |

   I left the remaining fields with their default values. Once you hit the bottom half of the form containing the "Loginname Attribute" field, click the "Preset Active Directory" button to populate all of the remaining fields. I've included a snapshot of what the config for my demo lab environment looked like:

   ![](/assets/img/posts/03/ex2.png)

   ![](/assets/img/posts/03/ex3.png)

   You can run "Quick Resolver Test" to test if a single user could be retrieved from LDAP, or "Test LDAP Resolver" to try and retrieve all of the users. Once you've gotten the pop up showing a successful retrieval, click save resolver.
6. Click the "Config" tab in the navigation bar at the top of the page, and then click on "Realms" in the navigation bar from within that page
7. Enter a name for your realm in the "new realm" textbox. In my case, I kept with the name "quickbreach". **Keep a note of the name of this realm, you will need it for part 5 step 4**.
8. Check the box of the ldapresolver on the right that you just created. When you are prompted for "priority", put 1.
9. Finally, click "Create Realm" on the right. (Note: In the screenshot below I actually created two different ldapresolvers, where as you may only see one)

![](/assets/img/posts/03/ex7.png)

***

## Part 3: Provide a User With a MFA Token

Now that privacyIDEA is linked to AD, lets assign a user a MFA token.

1. Navigate to the "Tokens" tab on the top left, and then click "Enroll Token"
2. Fill in the form until you reach the "Logginname Attribute" field. Here are the values used to create a Google Authenticator capable MFA token:
   * Select TOTP in the top drop down menu
   * Keep "Generate OTP Key on the Server" checked
   * Keep the OTP length at 6
   * Keep the timestep at 30 seconds
   * Select SHA1 for the hash algorithm
   * Description is optional
   * The "Realm" field should show the realm name from **part 2 step 7**. If it does not, then enter it.
   * In the "Assign token to user" field, begin typing the username of the user you wish to assign a token to. PrivacyIDEA will use the LDAP resolver(s) tied to the specified realm to find a matching user object in AD. Once you see your user in the drop down search, click them to select them.
   * If you enter a pin, then the user will need to enter their PIN+OTP when prompted for their OTP. I chose to leave mine blank so that my user only needs to enter their OTP.

   ![](/assets/img/posts/03/ex5.png)

   ![](/assets/img/posts/03/ex6.png)
3. Click "Enroll Token". You should then be brought to a screen like this:

   ![](/assets/img/posts/03/ex8.png)
   Scan the QR code provided with Google Authenticator or a similar app. This is the user's MFA secret.

You should now see this token in the "Tokens" tab of the navigation bar at the top. Now that we have a user paired with a MFA code, we need to hook this MFA solution up AD FS to handle MFA for Exchange.

***

## Part 4: Setup an AD FS Server

You now need to stand up a domain connected Windows Server to become the AD FS server. This server can either be it's own stand-alone Sever 2008/2012/2016 box, or it can be the same box as the Exchange CAS server. If this is a lab, you're probably fine just using the Exchange server - but for larger environments I would recommend a stand-alone server.

If a self-signed certificate was used for the privacyIDEA server, we need to install that certificate on the server becoming the AD FS server so that the privacyIDEA cert is trusted.

1. On the AD FS server (or the AD FS server to be), open Internet Explorer and navigate to https://_PISERVER_ and install the certificate. I have linked a good short guide on how to do that on [MSDN](https://blogs.technet.microsoft.com/sbs/2008/05/08/installing-a-self-signed-certificate-as-a-trusted-root-ca-in-windows-vista/)
2. We will also need a certificate for the AD FS server. If you have a CA that can issue a cert, then use that - otherwise here is an openssl one-liner to generate a self-signed PFX cert called "certificate.pfx"

       openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout server.key -out server.crt && openssl pkcs12 -export -out certificate.pfx -inkey server.key -in server.crt
3. Here is the heavy lifting: Follow steps 2 -> 4a from this [MSDN article](https://docs.microsoft.com/en-us/exchange/clients/outlook-on-the-web/ad-fs-claims-based-auth) to install the AD FS server role **but come back here once you finish step 4a!** Some of the steps in that article do not work entirely with privacyIDEA

***

**Notice:** After standing up the AD FS server role, make sure you create a DNS entry in your DC that matches the FQDN of the "Federation Service Name" from step 4 of 3b in the MSDN article. My AD FS server was installed on the same box as my exchange server (mail.quickbreach.com), so I manually created a record for adfs.quickbreach.com to point to the IP of the server.

***

If you followed the instructions of that article successfully, your AD FS Management pane should look like this
![](/assets/img/posts/03/ex9.png)

Now we need to add claims for both OWA and ECP.

1. Click the "Relying Party Trusts" folder on the left, and then right click the OWA trust you created in step 3 and click "Edit claim rules". Once the dialog opens, make sure you are in the "Issuance Transform Rules" tab.
2. At the bottom of that window, click "Add Rule"
3. In the "Claim rule template" drop down, select "Send claims using a custom rule" and hit next
4. In the "Claim rule name" box, enter "**privaceIDEA_UserSID**"
5. In the "Custom rule" box, copy and paste the below code

       c:[Type * "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer * "AD AUTHORITY"]
       => issue(store = "Active Directory", types = ("http://schemas.microsoft.com/ws/2008/06/identity/claims/primarysid"), query = ";objectSID;{0}", param = c.Value);
6. Click finish. You should now see the claim rule you just created in the "Issuance Transform Rules" pane. **You need to repeat this process** to create two more claim rules with the information below:

   Rule name: **privacyIDEA_ActiveDirectoryUPN**

       c:[Type * "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer * "AD AUTHORITY"]
       => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"), query = ";userPrincipalName;{0}", param = c.Value);

   Rule name: **privacyIDEA_GroupSID**

       c:[Type * "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer * "AD AUTHORITY"]
       => issue(store = "Active Directory", types = ("http://schemas.microsoft.com/ws/2008/06/identity/claims/groupsid"), query = ";tokenGroups(SID);{0}", param = c.Value);
7. You should now see a total of three rules in your "Issuance Transform Rules" pane as shown below

   ![](/assets/img/posts/03/ex10.png)
8. You can now close the "Edit Claim Rules for.." window. **You must now repeat steps 4 -> 10 for ECP**

***

**Notice:** This next step is more of a bugfix than anything I've read officialy, probably because I used self-signed certs, but I haven't gotten AD FS to work without it. I'm also unsure if this step should be executed on the AD FS server, or the Exchange server. In my case they were both the same machine, but your experience may vary.

***

1. We need to install the AD FS token signing certificate to the trusted root certificate authority. To accomplish this, open AD FS Management and on the left go to Service -> Certificates and then double click the "Token-Signing" certificate. Click install certificate, then select "local machine", then select "Place all certificates in the following store", click "Browse", and then finally select "Trusted root certification authorities". Click "Next" and then "Finish".
2. ![](/assets/img/posts/03/ex11.png)

***

## Part 5: Link AD FS to the PrivacyIDEA Server

Now that AD FS is up and running, we need to link it up with privacyIDEA. All of the steps below are being run from the AD FS server.

1. Download the [privacyIDEA-ADFS connector](https://github.com/sbidy/privacyIDEA-ADFSProvider/releases/download/1.2/privacyIDEAADFSProvider_V1.2.zip).  This guide used commit a096ffca29f75ef94ee74785c21547ca5dc39499
2. Extract everything to "C:\\Program Files\\privacyIDEAProvider"
3. You should now have a file located at "C:\\Program Files\\privacyIDEAProvider\\config.xml". Open it in notepad as an administrator
4. Apply the following changes in the config.xml file
   * Change "adfs" in the realm tags to the name of the realm you created on privacyIDEA in step 9 of part 2 earlier.
   * Change the URL in the url tags to that of your privacyIDEA instance
   * Change the ssl check from false to true

   Below is what my config.xml looked like

   ![](/assets/img/posts/03/moreEx.png)
5. Open Powershell as an administrator and run the following commands

       cd "C:\Program Files\privacyIDEAProvider\Install"
       
       .\Install-ADFSProvider.ps1
6. Pull up AD FS Management and then navigate to the "Authentication Policies" folder on the left. In the "Primary Authentication" section on the right, click "edit" underneath "Global authentication methods".
7. In the "Primary" tab of the window that just opened, ensure forms authentication is checked.

   ![](/assets/img/posts/03/ex12.png)
8. In the "Multi-Factor" tab, click the "add" button near the top next to the users/groups box and add the user we just assigned an MFA token. When this user is added, they will always be required to use their MFA token when logging in through AD FS. You do not need to check anything within the "Devices" section or the "Locations" section of this dialog.

A couple of notes:

***

* We edited the global authentication policy and required MFA from a specific user/group through that policy. This means that any time that user(s) authenticates through AD FS, they will be required to use their MFA code. You could, instead, only require MFA from a user when they're accessing  specific relying trusts. This is achieved through the "per relying trust" folder on the left and right clicking each endpoint to require it manually.
* For larger deployments, you would probably be better off creating an AD group for MFA users, and add that group in this dialog - rather than adding each individual user

***

1. Select "privacyIDEA-ADFSProvider_1.2.0.0" in the bottom of the "Multi-Factor" tab. Your setup should look similarly to the below screenshot:

   ![](/assets/img/posts/03/ex13.png)

We have now configured AD FS to talk to our privacyIDEA server to verify tokens, and which users should be required to provide a token. Now let's set up OWA/ECP to use AD FS for authentication - the final piece of the puzzle.

***

## Part 6: Configure OWA & ECP to use AD FS for Authentication

The last step is to instruct Exchange to force authentication to OWA and ECP through AD FS. You can either follow this guide, or you can finish up step 6 of this [MSDN Guide](https://docs.microsoft.com/en-us/exchange/clients/outlook-on-the-web/ad-fs-claims-based-auth)

1. On the AD FS server, run the following command to get the thumbprint for the AD FS token-signing certificate

       Set-Location Cert:\LocalMachine\Root; Get-ChildItem | Sort-Object Subject | where { $_.Subject -Like "ADFS Signing*" }
2. On the Exchange CAS server _MAILMAN_, log in as an Exchange administrator and open up the Exchange Management Shell
3. **CRITICAL:** Review the following commands, replace the FQDN for your CAS server, FQDN for your ADFS server, and the Thumbprint obtained in step 1, and then run them from within the Exchange Management Shell

       $uris = @(" https://CAS_SERVER_FQDN/owa/","https://CAS_SERVER_FQDN/ecp/")
       
       Set-OrganizationConfig -AdfsIssuer "https://ADFS_SERVER-FQDN/adfs/ls/" -AdfsAudienceUris $uris -AdfsSignCertificateThumbprint THUMBPRINT_FROM_STEP1

This was my setup:

    $uris = @(" https://mail.quickbreach.com/owa/","https://mail.quickbreach.com/ecp/")
    Set-OrganizationConfig -AdfsIssuer "https://adfs.quickbreach.com/adfs/ls/" -AdfsAudienceUris $uris -AdfsSignCertificateThumbprint 88970C64278A15D642934DC2961D9CCA5E28DA6B

1. Finally, the last step is to restart IIS services on the Exchange CAS server

       iisreset /noforce

At long last, we are done! You should now be able to navigate to your OWA, log in, and see a screen prompting you for your MFA token!

![](/assets/img/posts/03/ex14.png)

Remember: There are around 8 different ways to access a user's mailbox on Exchange, and **we have only protected 2 of them with MFA**. Organizations must require MFA protected Modern Authentication, or MFA protected Hybrid Modern Authentication, on their Exchange infrastructure to truly be covered.

Outlook Anywhere (aka. RPC/HTTP), POP3, and IMAP do not support Modern Authentication. Ensuring these endpoints are disabled is also strongly advised.