# Set up ActiveDirectory authentication in the Mage.ai Docker container

## Overview
A guide on how to set up Mage.ai to use ActiveDirectory authentication with PyODBC.

The guide is not completely agnostic and you may need to tailor some aspects to your  
or your company's specific requirements.

I am not an expert on neither Kerberos, nor ActiveDirectory, or Docker for that matter  
this guide just details the steps I took based on research and some internal help that  
worked for me.

I decided to share this guide here as I could not find anything on it during my  
research and it would have saved me a week's worth of work. 

Hopefully, it will help you.

## Kerberos
Kerberos is used to authenticate into ActiveDirectory. This is best achieved by  
using a Kerberos Sidecar Container. The login credentials need to be provided at  
start, then they are encoded in a `.keytab` file which is then used to authenticate  
into AD. If the password changes it is necessary to regenerate the `.keytab` file.  
The Sidecar Container is looping a bash script indefinitely which handles the auth.  
The script produces the tickets (credentials) that can be used by other Kerberos  
clients to authenticate into a service. The tickets are stored in a  
credential cache (ccache) which then can be shared accross different containers.

N.B.: I am using Kerberos and ActiveDirectory (AD) pretty much interchangably,  
but this guide is specifically for ActiveDirectory.

### Quick Rundown
We are going create a Kerberos Sidecar Container. In this container a shell script  
is going authenticate with AD and store the necessary credentials in the directory  
`/krb5/` with the filename `ccache`, as configured in the `krb5.conf` file under  
`default_ccache_name`. We are using the same `krb5.conf` file in every container,  
therefore other containers will look for the credentials in this directory as well.  
We are going to share the Kerberos Sidecar Container `/krb5/` directory with the  
host system to the `DOCK/kerb/krb5/cred/` directory. This is achieved by creating   
a volume bind in the `compose.yaml` file. Therefore, the credentials stored in the  
`ccache` file on the Sidecar will be available on the host system as well.   
From here the same volume bind method can be used to share the `ccache` file with  
other containers that require it - we need to bind the `DOCK/kerb/krb5/cred/`  
directory on the host system to the `/krb5/` directory on the container and,  
as long as the container has the same `krb5.conf` file in the  `/etc/` directory,  
it will automatically authenticate into AD using the shared `ccache`.

### Directory Structure
```
.
└── DOCK
	└── .env
	└── compose.yaml
	└── kerb
	│	└── Dockerfile
	│	└── rekinit.sh
	│	└── krb5
	│		└── conf
	│		│	└── krb5.conf
	│		└── cred
	│			└── svc.keytab (generated in the next step)
	│			└── ccache (generated on run)
	└── mage
		└── Dockerfile
```
### The .keytab File
To produce this file you need to have Kerberos installed on your system.  
I use Red Hat, but the Debian/Ubuntu version of these commands are also used  
when we are setting up Mage.ai to use the tickets we generate here.  
I will be using an example account name here: "svc-mage".  

1. Install Kerberos on your host system.  
  `dnf install -y krb5-workstation` 
2. Start the ktutil application.  
  `ktutil`
3. Generate a new .keytab from your credentials.  
  `addent -password -p <user> -k 1 -e RC4-HMAC -s "<user_seed>"`  
    * Replace `<user>` with account you will be using.  
      E.g.: svc-mage@REALM.DOMAIN.COM (no quotes)  
    * Replace `<user_seed>` with the AD specific seed.  
      Based on the `<user>` example this will be: "REALM.DOMAIN.COMsvc-mage" (with quotes)    
      This is CASE SENSITIVE so you might need to play around with it. Please see the first  
      Resource URL at the bottom of the page for more info.  
    * Final:  
      `addent -password -p svc-mage@REALM.DOMAIN.COM -k 1 -e RC4-HMAC -s "REALM.DOMAIN.COMsvc-mage"`
5. You will be prompted for the password. Input it.
6. Write the keytab into a .keytab file.  
  `write_kt <file_name>.keytab`
7. Request a ticket to test the keytab.  
  `kinit <user> -k -t <keytab_path>`  
    * Replace `<user>` with account you will be using - same account as in Step 3!
8. List the acquired tickets.  
  `klist`


### The krb5.conf File and Docker setup
This file is used to configure Kerberos. It lists how to handle different "realms"  
and which servers are used for authentication. You can also configure where you  
would like to store the authentication tickets. The original author used the  
`krb5.conf` file provided by their company. You can try to look for this in your Host  
system (the default path is `/etc/krb5.conf` OR the path is stored in the Environment  
variable `KRB5_CONFIG`) or get in touch with the relevant departments.  
Once you have the file you can copy it and use a modified version in your containers.  
In particular you need to modify the property `default_ccache_name` in your  
containers to a path that you can then bind to your host system in the 'Volumes'  
section of your `compose.yaml`. Be careful not to modify the `krb5.conf` file  
on your host system!

1. Create the directory tree as shown above. The `cred` folder will store your  
  credentials and should therefore be gitignored.
2. Create your `krb5.conf` file and set the `default_ccache_name` as in this exmaple  

```
[libdefaults]
	dns_lookup_kdc = false
	dns_lookup_real = false
	ticket_lifetime = 48h
	renew_lifetime = 7d
	forwardable = true
	rdns = false
	default_ccache_name = /krb5/ccache
	default_realm = REALM.DOMAIN.COM
[realms]
	REALM.DOMAIN.COM = {
		admin_server = server.realm.domain.com
		kdc = server.realm.domain.com
	}
	OTHER.DOMAIN.COM = {
		admin_server = ldap.server.com
		kdc = ldap.server.com
	}

```
3. Create the `rekinit.sh` file with the below contents. Remember to change <user>  
  to the account you used in Step 3 of the [.keytab section](#the-keytab-file).  
  This script will renew the tickets at regular intervals by basically repeating  
  Step 6 of the [.keytab section](#the-keytab-file). Make sure you replace `<user>`  
  with the same `<user>` as in the [.keytab section](#the-keytab-file).  
```
#!/bin/sh

[[ "$PERIOD_SECONDS" == "" ]] && PERIOD_SECONDS=3600

while true

do

# report to stdout the time the kinit was being run
 echo "*** kinit at "+$(date -I)
# run kinit 
 kinit <user> -V -k -t /krb5/svc.keytab 

# report the valid tokens to stdout
 klist

# sleep for the defined period, then repeat
 echo "*** Waiting for $PERIOD_SECONDS seconds"
 sleep $PERIOD_SECONDS
 done

```
4. Create the `Dockerfile` for the sidecar container.
```
FROM centos:centos7

# install the kerberos client tools

 RUN yum install -y krb5-workstation && mkdir /krb5 && chmod 755 /krb5
 
# add the keytab, the krb5 configuration and the rekinit script to the image

 ADD ./kerb/krb5/cred/svc.keytab /krb5/svc.keytab
 ADD ./kerb/krb5/conf/krb5.conf /etc/krb5.conf

 ADD ./kerb/rekinit.sh /
 RUN chmod +x /rekinit.sh
```
5. Add the `kerb` service to your docker compose yaml file.  
  In the volumes you are binding the `/kerb/krb5/cred` dir to the `/krb5/` dir  
  that you have created in the `Dockerfile` on the container. You have also added  
  to that directory the `.keytab` file you generated in [.keytab section](#the-keytab-file).  
  This way the contents of the `/krb5/` dir on your container are available on the  
  host system in the `kerb/krb5/cred/` directory. The `ccache` file where the  
  credentials are stored will be used by other containers to authenticate.    
  When the container is started it will immediately execute the `rekinit.sh` script  
  specified in the `command` property.  

```
services:
  kerb: 
    build:
      context: .
      dockerfile: kerb/Dockerfile
    volumes:
      - type: bind
        source: ./kerb/krb5/cred 
        target: /krb5/
    command: ["/bin/sh", "/rekinit.sh"]
```

## Mage.ai
The Mage.ai docker image needs to be slightly modified to include Kerberos  
when it is running. The same `krb5.conf` file will be used here as the one on the  
Kerberos Sidecar Container.

1. Create the Dockerfile with the below contents.  
```
FROM mageai/mageai:latest

# these arguments are passed from the compose.yaml file
# and are based on the contents of the .env file
 ARG USER_CODE_PATH
 ARG PROJECT_NAME

# install the kerberos client tools, create the credentials dir

 ENV DEBIAN_FRONTEND=noninteractive
 RUN apt-get update && apt-get install krb5-user -y && rm -rf /var/lib/apt/lists/* && mkdir /krb5 && chmod 755 /krb5

# add the same kerberos config file as used in the kerb image 
 
 ADD ./DOCK/kerb/krb5/conf/krb5.conf /etc/krb5.conf

# Copy the requiremens specified in the code to the image and install them 

 COPY requirements.txt ${USER_CODE_PATH}/requirements.txt 

 RUN pip3 install -r ${USER_CODE_PATH}/requirements.txt
```
2. Add the following service to your docker compose yaml.  
  PROJECT_NAME and ENV are defined in the `.env` file.  
  The `kerb/krb5/cred` dir which contains the updating credentials from the `kerb`  
  conainer is bound to the `/krb5/` directory on the `mage` container as read only.  
  The tickets are available on the `mage` container for AD authentication but  
  the `mage` container cannot make any changes to them. The `/krb5/` dir is also  
  specified in the `krb5.conf` as the location where it should look for valid  
  credentials. The container will be listening on the default HTTP port (80) and
  will forward incoming traffic to the default Mage.ai port (6789). When building
  image the `args` will be passed to the Dockerfile as defined in Step 1. The
  container is also set to be dependent on kerb, therefore kerb will be started  
  first and the `ccache` will be available to the mage container immediately.
  
```
services:
  mage:
    image: mageai/mageai:latest
    command: mage start ${PROJECT_NAME}
    env_file:
      - .env
    build:
      context: ..
      dockerfile: DOCK/mage/Dockerfile
      args:
        - USER_CODE_PATH=/home/src/${PROJECT_NAME}
        - PROJECT_NAME=${PROJECT_NAME}
    environment:
      USER_CODE_PATH: /home/src/${PROJECT_NAME}
      ENV: ${ENV}
    ports:
       - 80:6789
    volumes:
      - type: bind
        source: ./kerb/krb5/cred
        target: /krb5
        read_only: true
    restart: on-failure:5
    depends_on:
      - kerb
```
3. Use `docker compose up --build -d` in the directory where your compose yaml is.
4. Connect a shell to your Mage.ai container.  
  `docker exec -it mage_container_name sh`
5. Run `klist` inside the Mage container and verify the output. It should be like this  
  ```
Ticket cache: FILE:/krb5/ccache
Default principal: <user>

Valid starting     Expires            Service principal
02/09/24 20:58:45  02/10/24 06:58:45  krbtgt/SOME.REALM.SPECIFIC@SOME.REALM.SPECIFIC
        renew until 02/16/24 20:58:44
```
6. You can now use trusted connection in mage for your pyodbc connection strings when  
  connecting to AD authenticated SQL servers, like so:    
  `"DRIVER={ODBC Driver 18 for SQL Server}; SERVER=server_address; trusted_connection=yes; DATABASE=db_name; Encrypt=yes; TrustServerCertificate=yes"`
7. idk man, profit...


## Resources
* https://community.cloudera.com/t5/Community-Articles/quot-kinit-Preauthentication-failed-while-getting-initial/ta-p/346998#:~:text=The%20cause%20of%20the%20aforementioned,concatenated%20with%20the%20short%20name.
* https://askubuntu.com/questions/1017999/install-kerberos-client-without-interactive-session
* https://web.mit.edu/kerberos/krb5-1.12/doc/basic/ccache_def.html
* https://web.mit.edu/kerberos/krb5-1.12/doc/admin/conf_files/krb5_conf.html
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system-level_authentication_guide/configuring_a_kerberos_5_client
* https://www.redhat.com/en/blog/kerberos-sidecar-container
