# E-mail server setup.

## Depending if you already have an user for receiving mail for root add an user using command:

`sudo useradd -m <your_username>`

`sudo -u <your_username> mkdir -p /home/<your_username>/Maildir/{cur,new,tmp}`

## Install Postfix and mailutils

First step is to install  postfix and mailutils with a command:

`sudo apt-get install postfix mailutils`

There is a chance window will change to:

https://i.imgur.com/K93U91d.png

This is a configuration wizard which can help out to set up basic information about your e-mail service.
Press next on every stage without filling out details.


More in depth configuration of postfix with command:

`sudo dpkg-reconfigure postfix`

Press `ok` for the first screen.

https://i.imgur.com/K93U91d.png


Choose "Internet Site":

https://i.imgur.com/PKC56XD.png

Type in your domain name.
#### example.com

Type in name of your postmaster.
#### username

Next provide list of domains this machine should consider itself the final destination. 

#### leave default

Enforce synchronous updates on mail queue?

#### no

Local networks has not to be really changed.
#### leave default


Mailbox size limit to

#### 0

Local address extension character

#### +

 Internet protocols to use:

#### ipv4

Setting inet_protocols to ipv4 due error which happen to occur when Gmail Guidelines for IPv6 are not met.

Press enter and settings will be processed.

### Generate ssl certificate or add your own CA signed certificate
To generate ssl certificate and a key type in the terminal:

`openssl req -new -x509 -days 3650 -nodes -out /etc/ssl/certs/server.pem -keyout /etc/ssl/private/server.pem`

Set read and write permissions by:

`chmod o= /etc/ssl/private/server.pem`

### Modify main.cf
Modify main.cf using following commands:

`sudo postconf -e 'smtpd_tls_cert_file = /etc/ssl/certs/server.pem'`

`sudo postconf -e 'smtpd_tls_key_file = /etc/ssl/private/server.pem'`

`sudo postconf -e 'smtpd_tls_security_level = may'`

`sudo postconf -e 'smtpd_tls_auth_only = yes'`

`sudo postconf -e 'smtpd_tls_loglevel = 1'`

`sudo postconf -e 'smtp_tls_security_level = may'`

`sudo postconf -e 'smtp_tls_note_starttls_offer = yes'`

`sudo postconf -e 'smtpd_tls_received_header = yes'`

`sudo postconf -e 'home_mailbox = Maildir/'`

`sudo postconf -e 'mailbox_command = '`

`sudo postconf -e 'virtual_alias_maps= hash:/etc/postfix/virtual_alias'`

Assign email addresses to the users

Open/create file virtual with command:

`sudo nano /etc/postfix/virtual`

Type inside of the file e-mail address which will be moved to given user inbox:

`support@example.com <your_username>`

Ctrl + x to exit and y to approve changes

To apply changes in the postfix configuration:

`sudo postmap /etc/postfix/virtual`

`sudo /etc/init.d/postfix restart`

## Before you send an e-mail

Don’t send a test email to outlook, yahoo or AOL before setting up all the authentications within this guide.

Time for email blacklist re-evaluation can take up to 2 months

## Setting up SPF

Go to DNS provider of your domain and create new record set.

If you are using Amazon Web Services it will be Route 53.

Go to Hosted Zones, choose your domain and Create Record Set.

SET type to "MX" and type in value priority for your mail routing and the ip of your e-mail server:

`10 <ip_of_your_server>`

Save changes.

Create new record for SPF this time:

Set type to "TXT" and type in value:

`"v=spf1 mx a -all"`

Save changes.

## Setting up DMARC

Go to DNS provider of your domain and create new record set.

If you are using Amazon Web Services it will be Route 53.

Go to Hosted Zones, choose your domain and Create Record Set.

Set name to:

`_dmarc`

Set type to "TXT" and type in value:

`"v=DMARC1;p=quarantine;sp=quarantine;adkim=r;aspf=r"`

Save changes.

## Setting up DKIM

Install  opendkim with a command:

`apt-get install opendkim opendkim-tools`

Add Postfix and Opendkim user with a command:

`adduser postfix opendkim`

Remove  opendkim.conf file with a command:

`rm /etc/opendkim.conf`

Create opendkim.conf file with a command:

`nano /etc/opendkim.conf`

Paste the content:

    Syslog          yes

    UMask           002

    UserID          opendkim

    KeyTable        /etc/opendkim/key.table
    SigningTable        refile:/etc/opendkim/signing.table

    ExternalIgnoreList  /etc/opendkim/trusted.hosts
    InternalHosts       /etc/opendkim/trusted.hosts

    Canonicalization    relaxed/simple
    Mode            sv
    SubDomains      no
    AutoRestart     yes
    AutoRestartRate     10/1M
    Background      yes
    DNSTimeout      5
    SignatureAlgorithm  rsa-sha256

    OversignHeaders     From

Ctrl + x to exit and y to approve changes.

Create opendkim directory  with a command:

`mkdir /etc/opendkim`

Create `trusted.hosts` file with a command:

`nano /etc/opendkim/trusted.hosts`

Fill the content of the file:

    127.0.0.1
    ::1
    localhost
    192.168.0.0/255.255.255.0
    <ip_of_your_server>
    <your_domain>

Ctrl + x to exit and y to approve changes.

Create `signing.table` file with a command:

`nano /etc/opendkim/signing.table`

Fill the content of the file:

`*@<your_domain>   <your_domain>`

Ctrl + x to exit and y to approve changes.

Create `key.table` file with a command:

`nano /etc/opendkim/key.table`

Fill the content of the file:

`<your_domain>     <your_domain>:default:/etc/opendkim/keys/<your_domain>.private`

Ctrl + x to exit and y to approve changes.

Create directory for opendkim keys with command:

`mkdir -p /etc/opendkim/keys/<your_domain>`

Generate private and public key with command:

`opendkim-genkey -D /etc/opendkim/keys/<your_domain> -b 2048 -h rsa-sha256 -r -s default -d <your_domain> -v` 

Move and rename generated private key to `/etc/opendkim/keys` location defined in `key.table` with command:

`mv /etc/opendkim/keys/<your_domain>/default.private /etc/opendkim/keys/<your_domain>.private`

Set permissions on directory by:

chown -R opendkim:opendkim /etc/opendkim
chmod -R go-rw /etc/opendkim/keys

`chown -R opendkim:opendkim /etc/opendkim`

`chmod go-rw /etc/opendkim/keys`

Change directory to `/etc/opendkim/keys/` by:

`cd /etc/opendkim/keys/`

Set permissions on private key by:

`chmod 600 <your_domain>.private`

`chown opendkim:opendkim <your_domain>.private`

Restart opendkim with command:

`systemctl restart opendkim`

Check status of opendkim with command:

`systemctl status -l opendkim`

Display and copy content of the `default.txt` file with command:

`cat /etc/opendkim/keys/<your_domain>/default.txt`

This is the example output of the command:

    default._domainkey	IN	TXT	( "v=DKIM1; h=rsa-sha256; k=rsa; s=email; "
    	  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3Agxn5FsdbRsAarGxdyhYZgY11wWPlr+goV4P4CKMs2S0tHpquDZCgGOH/Tilc9laYT8l6kgh14IdouLRc3qVT/KJorWPRya8qUrgzQgN+hchzfn74hbLUmHPUPvtze6tDG4If2Zgf3MwAT8lMz2S74A1V4IGTZI7McwQd6yB7wiCGij7VrhC/HW+aeLrYgwBoxV7a8JMXtn3j"
    	  "pRoixnmJeoTfcEjHwx6a6rq7sXyVBV3/VTIHmbWtLGzGt1M2BtGJVZDEsNtHG/VB0FNF6b+FBK/yfgVcKyX40p+YHVTTJidVRtVjb3M2sbukQhurl6+a+CgjQn/GrrKJaiXQD0JQIDJQAB" )  ; ----- DKIM key default for example.com

Go to DNS provider of your domain and create new record set.

If you are using Amazon Web Services it will be Route 53.

Go to Hosted Zones, choose your domain and Create Record Set.

Set name to:

`default._domainkey`

Set type to "TXT" and type in value from your default.txt like by example below (remove all the new lines, otherwise it will produce error).

This is the example of the value:

`"v=DKIM1; h=rsa-sha256; k=rsa; s=email;""p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3Agxn5FsdbRsAarGxdyhYZgY11wWPlr+goV4P4CKMs2S0tHpquDZCgGOH/Tilc9laYT8l6kgh14IdouLRc3qVT/KJorWPRya8qUrgzQgN+hchzfn74hbLUmHPUPvtze6tDG4If2Zgf3MwAT8lMz2S74A1V4IGTZI7McwQd6yB7wiCGij7VrhC/HW+aeLrYgwBoxV7a8JMXtn3j""pRoixnmJeoTfcEjHwx6a6rq7sXyVBV3/VTIHmbWtLGzGt1M2BtGJVZDEsNtHG/VB0FNF6b+FBK/yfgVcKyX40p+YHVTTJidVRtVjb3M2sbukQhurl6+a+CgjQn/GrrKJaiXQD0JQIDJQAB"`

Save changes.

Test the setup with command (if it does not show any output then it is right):

`opendkim-testkey -d <your_domain> -s default`

There is a chance of the error occurring:

`opendkim-testkey: unknown hash 'rsa-sha256'`

For some reason `rsa` from `h=rsa-sha256;` has to be removed.

So the value of created record has to be changed to as on example below:

`"v=DKIM1; h=sha256; k=rsa; s=email;""p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA3Agxn5FsdbRsAarGxdyhYZgY11wWPlr+goV4P4CKMs2S0tHpquDZCgGOH/Tilc9laYT8l6kgh14IdouLRc3qVT/KJorWPRya8qUrgzQgN+hchzfn74hbLUmHPUPvtze6tDG4If2Zgf3MwAT8lMz2S74A1V4IGTZI7McwQd6yB7wiCGij7VrhC/HW+aeLrYgwBoxV7a8JMXtn3j""pRoixnmJeoTfcEjHwx6a6rq7sXyVBV3/VTIHmbWtLGzGt1M2BtGJVZDEsNtHG/VB0FNF6b+FBK/yfgVcKyX40p+YHVTTJidVRtVjb3M2sbukQhurl6+a+CgjQn/GrrKJaiXQD0JQIDJQAB"`

Save changes.

After the change and running again the test command:

`opendkim-testkey -d <your_domain> -s default`

No output should be produced and it means everything is fine.

Go to DNS provider of your domain and create new record set.

If you are using Amazon Web Services it will be Route 53.

Go to Hosted Zones, choose your domain and Create Record Set.

Set name to:
`_adsp._domainkey`

Set type to "TXT" and type in value:
`dkim=all`

Save changes.

### Connect opendkim to postfix

Create the opendkim socket directory with command:

`mkdir /var/spool/postfix/opendkim`

Set permission to the directory with command:

`chown opendkim:postfix /var/spool/postfix/opendkim`

Modify `opendkim` - configuration file with command:

`sudo nano /etc/default/opendkim`

Comment out any other `SOCKET=` which does not have `#` at the start of the line and append this one to the file:

`SOCKET="local:/var/spool/postfix/opendkim/opendkim.sock"`

Ctrl + x to exit and y to approve changes.

Modify `main.cf` postfix file with command:

`sudo nano /etc/postfix/main.cf`

Scroll down and append these lines:

    milter_protocol = 6
    milter_default_action = accept
    smtpd_milters = local:/opendkim/opendkim.sock
    non_smtpd_milters = local:/opendkim/opendkim.sock

Ctrl + x to exit and y to approve changes.

Restart opendkim and postfix with these commands:

`sudo /etc/init.d/postfix restart`

`sudo /etc/init.d/opendkim restart`


## Switch user and send an e-mail

Change user from root to `<your_username>` with command:

`su <your_username>`

Go to home directory:

`cd`

Send an e-mail with

`mail <target_e-mail>`

Enter and fill next values:

`CC: <can_be_left_empty>`

Subject: My first e-mail

Next after Subject is content of the e-mail, type in multiple lines.

To finish and send an e-mail follow up with

`~.`

## Removing additional information about sent e-mail:

There is a chance information when inspecting original e-mail will be shown like:

`(GNU Mailutils 2.99.99) were used to send this e-mail`

To avoid those kind of information being included in the e-mail there is solution.

Modify `main.cf` postfix file with commands:

`sudo postconf -e 'mime_header_checks = regexp:/etc/postfix/header_checks'`

`sudo postconf -e 'header_checks = regexp:/etc/postfix/header_checks'`

Create `header_checks` file with command:

`sudo nano /etc/postfix/header_checks`

Paste the content to the file:

    /^Received:.*with ESMTPSA/              IGNORE
    /^X-Originating-IP:/    IGNORE
    /^X-Mailer:/            IGNORE
    /^Mime-Version:/        IGNORE

Ctrl + x to exit and y to approve changes.

Load settings with command:

`postmap /etc/postfix/header_checks`

Reload postfix with:

`postfix reload`


## Mail Traffic
It's possible e-mail traffic on port 25 will be blocked by default on your server.
In this situation a ticket is required to the Vultr staff about it.
