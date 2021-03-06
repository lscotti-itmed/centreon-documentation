---
id: procedures
title: Procedures
---

This chapter describes advanced procedures for configuring your Centreon MAP
system.

## Monitoring your Centreon MAP server after installation

Centreon provides a plugin pack and a plugin to monitor your Centreon MAP
server.

### Installing the plugins

Use SSH to access the poller that will be monitoring your Centreon MAP server.
Install all the required plugins with the following commands:

    yum install centreon-pack-operatingsystems-linux-snmp
    yum install centreon-pack-applications-monitoring-centreon-map-jmx
    yum install centreon-pack-applications-webservers-tomcat-jmx

    yum install centreon-plugin-Operatingsystems-Linux-Snmp
    yum install centreon-plugin-Applications-Monitoring-Centreon-Map-Jmx
    yum install centreon-plugin-Applications-Webservers-Tomcat-Jmx

OPTIONAL - Only if you have a local MariaDB DB for your Map server

    yum install centreon-pack-applications-databases-mysql
    yum install centreon-plugin-Applications-Databases-Mysql

### Configure your database

Access your MariaDB server where the Centreon MAP database is stored (the Centreon
MAP database is called 'centreon\_studio' by default).

Run the following commands:

    mysql centreon_studio
    GRANT SELECT ON centreon_studio.* TO 'centreon_map'@'<POLLER_IP>' identified by 'PASSWORD';

  - Replace 'centreon\_studio' by the DB name of your Centreon MAP server.
  - Replace \<POLLER\_IP\> by the IP address of the poller which will be
    monitoring your DB.
  - Replace 'PASSWORD' by any password you prefer.

### Configure your services

Access your Centreon Web interface. Go to *Configuration \> Host \> Add*.

Fill in the basic information about your host and add the following host
templates:

  - OS-Linux-SNMP-custom
  - App-Monitoring-Centreon-Map-JMX-custom
  - App-Webserver-Tomcat-JMX-custom

Also add the following only if you have a local MariaDB DB on you Map server:

  - App-DB-MySQL-custom

![image](../assets/graph-views/map4-host-configuration.png)

Important:

1)  The above host templates are the three main templates required for
    monitoring your Centreon MAP server.

2)  The MySQL/MariaDB template is useful only if there is a MariaDB server on your
    Centreon MAP server (for Centreon MAP database).

3)  Enter the Jolokia URL, accessible on the Centreon Map server, through actuator endpoint:

    For an HTTP configuration

        http://<MAP_IP>:8080/centreon-studio/actuator/jolokia

    For an HTTPS configuration

        https://<MAP_IP>:8443/centreon-studio/actuator/jolokia

    > Replace \<MAP\_IP\> by the IP address of your Centreon MAP server.

4)  *If you have installed a MariaDB server on your Centreon MAP server*, enter
    the user/password you used in [Configure your
    database](#configure-your-database).

> Remember to check the "Create Services linked to the Template too" checkbox.

You can now export your configuration, and your Centreon MAP server will be
monitored.

![image](../assets/graph-views/map4-services.png)

> You may also just check the access to the following URL that tells that the server is Up or not 
> 
> - If your server is running in http mode: http://<IP_SERVER\_MAP>:8080/centreon-studio/api/beta/actuator/health.
> - If your server is running in HTTPS mode: https://<IP_SERVER_MAP>:8443/centreon-studio/api/beta/actuator/health.

## Migrating your Centreon MAP server

This section explains how to move Centreon MAP server over to another server.
This task may be useful if you need to migrate your reporting server from CentOS
6 to CentOS 7.

### Install the new Centreon MAP server

Please refer to the installation chapter in this documentation to install your
new Centreon MAP server.

### Synchronize the data

Stop Centreon Map service on **both** Centreon MAP servers:

    systemctl stop centreon-map

Dump the Centreon MAP data:

    mysqldump -u XXXXXX -p centreon_studio > /tmp/centreon_studio.sql

Upload *centreon\_studio.sql* to the new Centreon MAP (in /tmp) server and
import it into the database:

    mysql -u XXXXXX -p centreon_studio < /tmp/centreon_studio.sql

Start Centreon Map service on the new Centreon MAP servers:

    systemctl start centreon-map

## Centreon MAP configuration files

> We advise you against editing the configuration files manually unless you are
> an experienced user.

The four configuration files are located in */etc/centreon-studio/*. Their
templates can be found in */etc/centreon-studio/templates/*.

The configuration script replaces the macros in these templates and copies them
to the folder /etc/centreon-studio.

If these files are modified, the server must be restarted with the command:

    systemctl restart centreon-map

> Do not delete any variables in these files\! This may cause the server to
> malfunction or not to start up.

## HTTPS/TLS Configuration

### HTTPS/TLS configuration with a recognized key

> This section describes how to add a **recognized key** to the Centreon Map server.
> If you want to create an auto-signed key and add it to your server, please
> refer to the [following section](#httpstls-configuration-with-an-auto-signed-key)

You will require:

- A key file, referred to as *key.key*.
- A certificate file, referred to as *certificate.crt*.

Access the Centreon MAP server through SSH.

Create a PKCS12 file with the following command line:

    openssl pkcs12 -inkey key.key -in certificate.crt -export -out keys.pkcs12

Then, import this file into a new keystore (a Java repository of security certificates):

    keytool -importkeystore -srckeystore keys.pkcs12 -srcstoretype pkcs12 -destkeystore studio.jks

Put above keystore file (studio.jks) to the folder "/etc/centreon-studio/", and set below parameters inside "/etc/centreon-studio/studio-config.properties"

    centreon-map.keystore=/etc/centreon-studio/studio.jks
    centreon-map.keystore-pass=xxx

> Replace the keystore-pass value "xxx" with the password you used for 
> the keystore and adapt the path (if it was changed) to the keystore.

### HTTPS/TLS configuration with an auto-signed key

> Enabling the TLS mode with an auto-signed key will force every user to add an 
> exception for the certificate before using the web interface. 
> Enable it only if your Centreon also uses this protocol.
> Users will have to open the URL https://centreon-map-url:8443/centreon-studio/api/beta/actuator/health.
> *The solution we recommend is to use a recognized key method, as explained above.*

On the Centreon MAP server

Create a keystore.

Go to the folder where Java is installed:

    cd $JAVA_HOME/bin

Then generate a keystore file with the following command:

    keytool -genkey -alias studio -keyalg RSA -keystore /etc/centreon-studio/studio.jks

The alias value "studio" and the keystore file path /etc/centreon-studio/studio.jks may be changed, but unless there is a specific reason, we advise keeping the default values.

Provide the needed information when creating the keystore.

At the end of the screen form, when the "key password" is requested, use the same password as the 
one used for the keystore itself by pressing the ENTER key.

Put above keystore file (studio.jks) to the folder "/etc/centreon-studio/", and set below parameters
inside "/etc/centreon-studio/studio-config.properties"

    centreon-map.keystore=/etc/centreon-studio/studio.jks
    centreon-map.keystore-pass=xxx

> Replace the keystore-pass value "xxx" with the password you used for the keystore 
> and adapt the path (if it was changed to the keystore.

### Activate TLS profile of Centreon MAP service

1) Stop Centreon MAP service:
``` shell
systemctl stop centreon-map
```
2) Edit the file "/etc/centreon-studio/centreon-map.conf", adding ",tls" after "prod" profile
``` shell
RUN_ARGS="--spring.profiles.active=prod,tls"
```
3. Restart Centreon MAP service.
``` shell
systemctl start centreon-map
```
Centreon MAP server is now configured to respond to requests from HTTPS at port 8443.
For the requirement of changing service's port, refer to :ref:`change_server_port`

> Don't forget to modify the URL on Centreon side in 
> **Administration** > **Extensions** > **Map** > **Options** => **Map server address** 

## Broker configuration

An additional broker output for Centreon central (centreon-broker-master) has
been created during the installation. You can check it in your central Centreon
web interface *Centreon \> Configuration \> Pollers \> Broker Configuration* in
the Main menu \> Centreon-broker-master.

The output configuration should look like this:

![image](../assets/graph-views/output_broker.png)

### Setup TLS connection between broker and map server

The socket connection between broker and map server can be secured by using TLS
protocol.

#### Broker side configuration

You could enable TLS output and set up broker's private key and public
certificate as described in [broker TLS
output](https://documentation.centreon.com/docs/centreon-broker/en/latest/user/modules.html#tls)

![image](../assets/graph-views/output_broker_tls.png)

> "Trusted CA's certificate" field is optional. If you activate broker's client
> authentication by setting this "ca\_certificate.crt", then you must setup a
> [keystore for map server](#httpstls-configuration)

> You MUST push the new broker configuration and restart the broker after
> configuration.

#### Map server side configuration

First of all, you should `activate HTTPS/TLS of Centreon MAP service <tls_configuration.html>`_

Than, set the following parameter in map server configuration at “/etc/centreon-studio/studio-config.properties“ to enable TLS socket connection with broker :

    broker.tls=true

**1. Self signed broker certificate**

If the broker public certificate is self signed, you must create a trust store
containing given certificate or its CA certificate with the following command
line:

    keytool -import -alias centreon-broker -file broker_public.crt -keystore truststore.jks

  - "broker\_public.crt" is broker public certificate or its CA certificate in
    PEM format.
  - "truststore.jks" is the generated trust store in JKS format.
  - a store password is required during generation

Then, put the generated output file "truststore.jks" into "/etc/centreon-studio"
of map server host.

and add truststore parameters in - “/etc/centreon-studio/studio-config.properties“

    centreon-map.truststore=/etc/centreon-studio/truststore.jks
    centreon-map.truststore-pass=XXXX

> Replace the trustStorePassword value "xxx" with the password you used when
> generate the trust store

Meanwhile, you should activate the "tls_broker" profile of Centreon MAP service:
Edit the file  "/etc/centreon-studio/centreon-map.conf", replace ",tls" by ",tls_broker" after "prod" profile

    RUN_ARGS="--spring.profiles.active=prod,tls_broker"

> "tls_broker" profile implies "tls" profile. So Centreon MAP service serves necessarily HTTPS.


**2. Recognized CA signed broker certificate**

If the broker public certificate is signed with a recognized CA, the JVM default trust store "cacerts (/etc/pki/java/cacerts)" will be used. Nothing to configure for Centreon MAP service

## Backup of Centreon MAP server

### Saved items

The saved items are:

  - Saving configuration files (**/etc/centreon-studio**)
  - Saving database **centreon\_studio**

### How it works ?

The backup script is executed on a daily basis (2AM) with a cron job located in
**/etc/cron.d/centreon-map-server-backup**:

    #
    # Cron to backup Centreon MAP server
    #
    PATH=/sbin:/bin:/usr/sbin:/usr/bin

    # rewrite file with new cron line
    CRONTAB_EXEC_USER=""

    0 2 * * * root bash /usr/share/centreon-map-server/bin/centreon-map-server-backup.sh >> /var/log/centreon-studio/backup.log 2>&1

The backup **centreon-map-server-yyyy-mm-dd.tar.gz** is stored in
**BACKUP\_DIR**, which is defined in configuration file.

### Backup parameters

Backup parameters are stored in **/etc/centreon-studio/backup.conf**

  - ENABLE: enable/disable backup mechanism (default value: 0)
  - BACKUP\_DIR: where the backup is stored (default value: /var/backup)
  - RETENTION\_AGE: backup retention in days (default value: 8)

> **We advise to export backups to another resource in order to secure them.**

### Restore data from Centreon MAP server

Restore process is divided in several steps:

  - Extracting backup
  - Restoring configuration files
  - Restoring database

> **We assume that you have followed the Centreon MAP server installation
> procedure to get a fresh install.**

### Extracting backup

Get the last **centreon-map-server-yyyy-mm-dd.tar.gz** backup and extract it
into **/tmp** directory:

    cd /tmp
    tar xzf centreon-map-server-yyyy-mm-dd.tar.gz

### Restoring configuration files

To restore configuration files, run the following command:

    cp -R etc/centreon-studio/* /etc/centreon-studio/

### Restoring database

To restore **centreon\_studio** database, run the following command:

    systemctl stop centreon-map
    mysql -h <db_host> -u <db_user> -p<db_password> <db_name> < centreon-map-server.dump
    systemctl start centreon-map

## Change Centreon Map server port

By default, the Centreon MAP server is listening and sending information through
the port 8080. If you set the SSL (see [HTTPS/TLS
Configuration](#httpstls-configuration), use the port 8443.

You can change this port (e.g., if you have a firewall on your network blocking
these ports).

> If the new port is below 1024, use this procedure below "Define port below 1024" instead.

On your Centreon MAP server, stop the Centreon Map server:

    systemctl stop centreon-map

Edit the studio-config.properties settings file located in /etc/centreon-studio:

    vim /etc/centreon-studio/studio-config.properties

Add the following line at the MAP SERVER section

    centreon-map.port=XXXX

...replace *XXXX* with the port you want.

Then restart the Centreon Map server:

    systemctl start centreon-map

Wait for Centreon MAP service to start completely (~30 sec to 1 minutes). Test that your server is
up and accessible on the new port you defined by entering the following URL in
your web browser:

    http://\<IP\_MAP\_SERVER\>:\<NEW\_PORT\>/centreon-studio/api/beta/actuator/health

## Define port below 1024

You may want to setup your server to listen and send data through ports below
1024, such as port 80 or 443 (as these ports are rarely blocked by a firewall).

If you want to set a port below 1024, the method is different since all ports
under 1024 are restricted and only accessible through special applications.

There are a few different workarounds for this issue. One method is "port
forwarding" through the firewall.

> For this example, set the MAP server to listen and send data through port 80.
> Replace each occurence of *80* with the port you want to use.

1.  Check your firewall.

On your MAP server, run the following command to check that the firewall is
running:

    # systemctl status iptables
    Table: raw
    Chain PREROUTING (policy ACCEPT)
    num  target     prot opt source               destination

    Chain OUTPUT (policy ACCEPT)
    num  target     prot opt source               destination

    Table: mangle
    Chain PREROUTING (policy ACCEPT)
    num  target     prot opt source               destination
    ...
    ...
    ...

If your firewall is stopped, you will see the following output:

    iptables: Firewall is not running.

Start the firewall:

    systemctl start iptables

2.  Enable a connection on the port for MAP for listening and sending.

Execute the following lines on your console:

    /sbin/iptables -A OUTPUT -p tcp --dport 80 -j ACCEPT
    /sbin/iptables -A INPUT -p tcp --dport 80 -j ACCEPT

3.  Add port forwarding.

Execute the following line on your console:

    iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

4.  Restart and save.

Restart your firewall:

    systemctl restart iptables

Save this configuration so it will be applied each time you reboot your server:

    /sbin/iptables save

Your Centreon MAP server is now accessible on port 80. Check this by entering
the following URL in your browser:

http://\<IP\_MAP\_SERVER\>/centreon-studio/api/beta/actuator/health

You should see server's state:
``` shell
{"status":"UP"}
```
> Don't forget to update both your desktop client configuration and your web
> interface configuration. For your desktop client, follow the instructions for
> setting up a server connexion `here <qc_login>`
> For your web interface, follow these instructions: `these
> steps<web_configuration>`
