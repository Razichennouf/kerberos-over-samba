# Kerberos_Over_smb
<pre>
<b>What is SMB :</b> Samba is a suite of applications that implements the Server Message Block (SMB) protocol. Many operating systems, including Microsoft Windows, use     the SMB protocol for client-server networking. Samba enables Linux / Unix machines to communicate with Windows machines in a network.
<b>What is Kerberos :</b> Kerberos uses symmetric-key cryptography to authenticate users to network services, which means passwords are never actually sent over the network.

<h1> Setting up SAMBA server </h1>
   <b>Create a Samba user account for each user who will be accessing the shared resources on the Linux machine. You can use the smbpasswd command to do    this. For example:</b>
   $ useradd smbuser
   $ smbpasswd -a smbuser =>  System_user is an aleady created user on the samba server.
   <b>Preferred to create a user for samba sharing and isolate it with least previleges</b>
   $ sudo apt install samba -y
   <b>Allow firewall rules on linux server</b>
   $ sudo ufw app list
   $ sudo ufw allow samba
   $ sudo systemctl enable --now smbd
   $ sudo smbpasswd -a ubuntu-desk
   $ sudo vim /etc/samba/smb.conf
     [samba-share1]
         path = /home/samba/samba-share1
         read only = no
         guest ok = yes
     [samba-share]
         path = /home/samba/samba-share
         read only = no
         guest ok = yes
         writable = yes
         guest ok = yes
         guest only = yes
         read only = no
         create mode = 0777
         directory mode = 0777
         force user = nobody
         #Kerberos setup
         security = ads
         realm = MESSENGER.COM
         password server = messenger.com
         
      $ sudo systemctl restart smbd
      <b>Create sharing folder</b>
         sudo mkdir -p /home/samba/samba-share
         sudo chmod 777 /home/samba/samba-share
         sudo mkdir -p /home/samba/samba-share1
         sudo chmod  777 /home/samba/samba-share1
 => To test access SAMBA server you have to use 2 backslashes : \\192.168.233.45
	<img src="https://github.com/Razichennouf/kerberos-over-samba/blob/main/images/Samba%20share%20built%20in%20authentication.PNG"
	width="400" />

<h1> Setting up Kerberos KDC,PDC,Pricipal names, Schemas/databeses  </h1>   
<b>Setup Kerberos KDC , realm (Kerberos server) , administrative server</b>
<b>What is realm</b> :
    What's a Kerberos Realm? A Kerberos realm is the domain over which a Kerberos authentication server has the authority to authenticate a user, host or service. A realm name is often, but not always the upper case version of the name of the DNS domain over which it presides
 1) First we should update and check the <b>time zones</b> if they are synchronized 
    $ timedatectl
 2) Domain name system
    $ nano /etc/hosts and add the corresponding Ip adresses of the two machines
      192.168.233.108 messenger.com kdc "Server"
 4) Install kerberos on Server and tools such as kadmin ect..
      $ sudo apt update & apt upgrade
      $ sudo apt install krb5-kdc krb5-admin-server krb5-config
    In case of a problem you could 
      $ apt remove
      $ apt purge the modules
    Setup 
      DNS :
        A page will trigger <b>realm</b> you should enter your DNS in UpperCase thats what we call a REALM
      Realm :
        MESSENGER.COM
	<img src="https://github.com/Razichennouf/kerberos-over-samba/blob/main/images/Kerberos%20Setup%20Realm%20DNS.PNG"
	width="600" />
      Administrative Server : 
        messenger.com <= same server
	<img src="https://github.com/Razichennouf/kerberos-over-samba/blob/main/images/Administrative%20server.PNG"
	width="600"/>
      Edit config file :
        nano /etc/krb5.conf
         [libdefaults]
	      default_realm = MESSENGER.COM
	      default_pricipal_name=root/admin
         [realms]
            MESSENGER.COM = {
               kdc = messenger.com
               admin_server = messenger.com
            }

         [domain_realm]
               messenger.com = MESSENGER.COM
              .messenger.com= MESSENGER.COM
   5) Enable kerberos services :
         systemctl status krb5-kdc => "Check KDC service"
         systemctl start krb5-kdc => "Start KDC service"
         systemctl start krb5-admin-server.service => "Start KADMIN utility"
         
   6) This script should be run on the master KDC/admin server to initialize a Kerberos realm.  
   It will ask you to type in a master key password. This password will be used to generate a key that is stored 
   in /etc/krb5kdc/stash.
      => "krb5_newrealm"      
   7) Creating kerberos schema or we say database :
      => "kdb5_util create -s"
	   => KDC database master key => "tekup"
   
   7) In order to create <b>Principals</b> in "Kadmin server" first user "kadmin.local" and adding a <b>Keytab</b>
       1) kadmin.local -q "addprinc user1/admin"
            kadmin.local -q "list_principals"
            "OR"
            configure your database : sudo kadmin.local	
            "kadmin.local" : addprinc admin
            "kadmin.local" : ktadd admin
            "kadmin.local" : quit
	    => Adding Principal
	   <img src="https://github.com/Razichennouf/kerberos-over-samba/blob/main/images/Configuring%20database%20adding%20principal.PNG"
	   width="600"/>
	    =>Adding a key tab
	   <img src="https://github.com/Razichennouf/kerberos-over-samba/blob/main/images/Adding%20a%20keytab%20for%20principal.PNG"
	   width="600"/>
       2) Use kadmin tool 
            Fix error you are getting =>"kadmin: Client 'root/admin@MESSENGER.COM' not found in Kerberos database while initializing kadmin interface"
               [realms]
                  EXAMPLE.COM = {
                  krbtgt_principal = krbtgt/EXAMPLE.COM@EXAMPLE.COM
                  }
      =>"The main difference" between kadmin and kadmin.local is that kadmin connects to the Kerberos database over the network,
         while kadmin.local connects to the database directly on the local system
       
       =>Checking if there is a key Obtained from the <b>Key distribution center</b>
     		<img src="https://github.com/Razichennouf/kerberos-over-samba/blob/main/images/Testing%20KDC%20and%20checking%20if%20there%20is%20a%20key%20obtained.PNG"
		width="600"/>
	3) Automate principale creation With 
		- Via Python script :
			import subprocess
			# Define the user information
			username = "testuser"
			password = "TestPasswd1"
			realm = "EXAMPLE.COM"

			# Create the user account
			subprocess.run(["kadmin.local", "-q", f"addprinc -pw {password} {username}@{realm}"])

			# Add the user to the appropriate Kerberos group
			subprocess.run(["kadmin.local", "-q", f"modprinc -addgroup {username} {realm}"])
		- Via Bash script : 
			#!/bin/bash
			# Read the list of users from the file
			while read username; do
			  # Create a new principal for the user
			  echo "Creating principal for $username"
			  kadmin.local addprinc $username
			done < users.txt
			I)To use this script, save it to a file and make it executable using the chmod command:
		chmod +x create_principals.sh
			II)Then, create a file called users.txt that contains a list of the users that you want to create principals for, with one username per line. For example:
				user1
				user2
				user3
			III) Finally, run the script to create the principals:
				./create_principals.sh
       8) Configure the PDC  
       9) Check the Kadmin service is loaded & check the file kadm5.acl if exists else : 
       	 => journalctl -xeu krb5-admin-server.service 
	    touch /etc/krb5kdc/kadm5.acl
         => add the HOST to "REALM"
		   => kadmin
  
  
That's it! Your Linux machine should now be set up to share the samba-share and samba-share1 directories with other computers on the network using Samba. Other computers on the network should be able to access these directories by connecting to the Linux machine using the <b>Kerberos protocol<b/> Over Samba protocol (e.g., \\LINUX_MACHINE\documents). 
=> Login to your samba share wrapped with kerberos.
<img src="https://github.com/Razichennouf/kerberos-over-samba/blob/main/images/Samba%20share%20built%20in%20authentication%202.PNG" 
	width="400"/>
</pre>
