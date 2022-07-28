# Describes adapted build using shared folders and scripts to manually build islandora.

### Notes

This build is adapted from this link and it may help to read it or follow along:
- https://islandora.github.io/documentation/installation/manual/introduction/
- some downloads will need new links as new versions come out
- many config files and scripts are included to elimanate a lot of typing within the build. 
- you'll need to enable shared folders in the vm settings and place the files in the shared folder
- code outside a comment or gui instruction is indended to be executed in the the vmware terminal, ie:
- ```whoami```
- you should know the linux command line fairly well
- you should know how to use nano or some other command line editor vi/m emacs
- you need to have downloaded vmware, and an ubuntu 20.04 server image.

**Pre-BUILD Requirements**

### download vmware
- https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html
- LSU has a way to get a liscense: https://software.grok.lsu.edu/Article.aspx?articleId=20512
- visit LSU box and download shared folder contents these will be your configuration files and others needed for the build add them to your shared folder in the vmware machine
- https://lsu.box.com/s/o2uy4yx57lrzlu6tbpmuis6n5ocgq1r9

- create a vmware machine 
- install ubuntu 20.04 Server from an ISO image:
- https://ubuntu.com/download/server#downloads
- 20 GB on one file for disk size 

- make sure the vm is on bridged network setting for installation
- (right click the vm, go to settings>network adapter, select "Bridged", then save)

- go through the os installation process
- all prompts you don't need to change anything unless you want openssh (for remote acces via ssh)
- set your username and password
- allow the OS to install

- when the installation finishes accept the prompt to reboot the machine

- when the vm boots up the os log in with you username and password you set before

## any command that requires downloading (apt, wget, git clone etc) will require you to set the vm in bridged network mode (to get outside data)

- try the "vmware-netcfg" command on the host machine if you have trouble connecting. 
- I have found swapping between bridged and host-only connections are sometimes required

- check your network connection in the command line with 
- ```ping www.google.com```

- if you get bytes back you're connection is good.

## enable shared folders on the vmware instance
- (toggle "Connected" button)
- (right click the vm, settings, options, shared folders, select "Always Enabled")
- (select a path to the files' folder, click save)
- I keep my path simple and put the files in a folder called 'shared'
- my path is /mnt/hgfs/shared within the vm, if you use a different path, change it in all commands that use '/mnt/hgfs/shared'

**Begin Build**

These commands should all be executed in sequence:

- ```sudo apt -y update```
- ```sudo apt -y install apache2 apache2-utils```
- ```sudo a2enmod ssl```
- ```sudo a2enmod rewrite```
- ```sudo systemctl restart apache2```
- ```sudo usermod -a -G www-data `whoami` ```
- ```sudo usermod -a -G `whoami` www-data```

Log out of the vm (CTL-D) (this is neccessary for group settings to be applied)
Log back in

Double check your shared folder exists

- ```ls /mnt/hgfs/shared```


NOTE

(if you don't see all the files or try:
- ```sudo vmhgfs-fuse .host:/ /mnt/hgfs/ -o allow_other -o uid=1000```


- ```sh /mnt/hgfs/shared/scratch_2.sh```

the above command runs the following script (which is tedious to type, there are some copy paste limitations in vmware when using alt-keyboard layouts):
- ```sudo apt -y install php7.4 php7.4-cli php7.4-common php7.4-curl php7.4-dev php7.4-gd php7.4-imap php7.4-json php7.4-mbstring php7.4-opcache php7.4-xml php7.4-yaml php7.4-zip libapache2-mod-php7.4 php-pgsql php-redis php-xdebug unzip postgresql```

Edit the postgresql.conf file starting at line 642

- ```sudo nano +642 /etc/postgresql/12/main/postgresql.conf```

(if you don't like nano use a different editor vi/vim/emacs )

change line 642 from 
>```
>bytea_output 'hex'
>```

change to
>```
>bytea_output 'escape'
>```

(CTL+o)(to save)(CTL+x to close)

- ```sudo systemctl restart postgresql```

- ```sudo apt install curl git```

- ```sh /mnt/hgfs/shared/scratch_3.sh```

#scratch_3.sh runs:
>```
>#!/bin/bash
>curl "https://getcomposer.org/installer" > composer-install.php
>chmod +x composer-install.php
>php composer-install.php
>sudo mv composer.phar /usr/local/bin/composer
>sudo mkdir /opt/drupal
>sudo chown www-data:www-data /opt/drupal
>sudo chmod 775 /opt/drupal
>sudo chown -R www-data:www-data /var/www/
>git clone https://github.com/drupal-composer/drupal-project.git
>```



- ```sh /mnt/hgfs/shared/scratch_4.sh```

scratch_4.sh runs:

>```
>cd drupal-project
># Expect this to take a little while, as this is grabbing the entire
># requirements set for Drupal.
>sudo -u www-data composer create-project drupal-composer/drupal-project:9.x-dev /opt/drupal --no-interaction
>sudo ln -s /opt/drupal/vendor/drush/drush/drush /usr/local/bin/drush
>```

-  ```sudo nano /etc/apache2/ports.conf```

remove everything but (CTL+k)(to yank lines out of the file)

> ```Listen 80```

save the file (CTL+o) then (CTL+x)

Edit the apache 000-default.conf file

- ```sudo nano /etc/apache2/sites-enabled/000-default.conf```

edit file to contain only the following:
>```
><VirtualHost *:80>
>  ServerName localhost
>  DocumentRoot "/opt/drupal/web"
>  <Directory "/opt/drupal/web">
>    Options Indexes FollowSymLinks MultiViews
>    AllowOverride all
>    Require all granted
>  </Directory>
>  ErrorLog "/var/log/apache2/localhost_error.log"
>  CustomLog "/var/log/apache2/localhost_access.log" combined
></VirtualHost>
>```

save (CTL+o) exit(CTL+x)

in future when ssl is needed for prod build:
- https://www.digitalocean.com/community/tutorials/how-to-install-an-ssl-certificate-from-a-commercial-certificate-authority

- ```sudo systemctl restart apache2```

- ```sudo -u postgres psql```

#from within the postgres cli:
>```
>create database drupal9 encoding 'UTF8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8' TEMPLATE template0;
>create user drupal with encrypted password 'drupal';
>grant all privileges on database drupal9 to drupal;
>#\q to quit
>\q
>```

install a drupal

- ```cd /opt/drupal/web```
- ```sudo drush -y site-install standard --db-url="pgsql://drupal:drupal@127.0.0.1:5432/drupal9" --site-name="LDL 2.0" --account-name=islandora --account-pass=islandora```

next install tomcat and cantaloupe

- ```sudo apt -y install openjdk-11-jdk openjdk-11-jre```

- ```update-alternatives --list java```

the above should output something like "/usr/lib/jvm/java-11-openjdk-amd64/bin/java"

note this path for later use as JAVA_HOME

note it's the path above without /bin/java:
ie:
"/usr/lib/jvm/java-11-openjdk-amd64"

- ```sudo addgroup tomcat```
- ```sudo adduser tomcat --ingroup tomcat --home /opt/tomcat --shell /usr/bin```

choose a password
ie: password: "tomcat"
press enter for all default user prompts
type y for yes

find the tar.gz here: https://tomcat.apache.org/download-90.cgi
copy the TOMCAT_TARBALL_LINK

watch for version change
- ``` cd /opt```
- ```sudo wget -O tomcat.tar.gz TOMCAT_TARBALL_LINK```
- ```ls /opt``` to find name of TOMCAT_DIRECTORY: (recently it was: apache-tomcat-9.0.65)
- ```sudo tar -zxvf tomcat.tar.gz```
- ```sudo mv /opt/TOMCAT_DIRECTORY/* /opt/tomcat```
- ```sudo chown -R tomcat:tomcat /opt/tomcat ```

- ```sh /mnt/hgfs/shared/scratch_5.sh```

scratch_5.sh will run the following file:
>```
>#!/bin/bash
>cd /opt
>#O not 0
>sudo wget -O tomcat.tar.gz https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.65/bin/apache-tomcat-9.0.65.tar.gz
>sudo tar -zxvf tomcat.tar.gz
>#don't miss the star*
>sudo mv /opt/apache-tomcat-9.0.65/* /opt/tomcat
>sudo chown -R tomcat:tomcat /opt/tomcat
>```

- ```sh /mnt/hgfs/shared/scratch_6.sh```

scratch_6 runs this file:

>```
>sudo cp /mnt/hgfs/shared/setenv.sh /opt/tomcat/bin/
>sudo chmod 755 /opt/tomcat/bin/setenv.sh
>sudo cp /mnt/hgfs/shared/tomcat.service /etc/systemd/system/tomcat.service 
>sudo chmod 755 /etc/systemd/system/tomcat.service
>#check your cantaloupe version 
>sudo wget -O /opt/cantaloupe.zip https://github.com/cantaloupe-project/cantaloupe/releases/download/v5.0.5/cantaloupe-5.0.5.zip
>sudo unzip /opt/cantaloupe.zip
>sudo mkdir /opt/cantaloupe_config
>sudo cp cantaloupe-5.0.5/cantaloupe.properties.sample /opt/cantaloupe_config/cantaloupe.properties
>sudo cp cantaloupe-5.0.5/delegates.rb.sample /opt/cantaloupe_config/delegates.rb
>sudo touch /etc/systemd/system/cantaloupe.service 
>sudo chmod 755 /etc/systemd/system/cantaloupe.service
>sudo cp /mnt/hgfs/shared/cantaloupe.service /etc/systemd/system/cantaloupe.service
>sudo systemctl enable cantaloupe
>sudo systemctl start cantaloupe
>```

check that unzip step worked ``` ls /opt/cantaloupe``` or something...

note: you may need this command if like me you had to edit cantaloupe.service several times: 
- ```sudo systemctl daemon-reload```

=======================================

## Installing fedora

- ```sudo systemctl stop tomcat```
- ```sudo mkdir -p /opt/fcrepo/data/objects```
- ```sudo mkdir /opt/fcrepo/config```
- ```sudo chown -R tomcat:tomcat /opt/fcrepo```

- ```sudo -u postgres psql```

>```
>create database fcrepo encoding 'UTF8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8' TEMPLATE template0;
>create user fedora with encrypted password 'fedora';
>grant all privileges on database fcrepo to fedora;
>\q
>```

first check that you mnt point is still working

- ```ls /mnt/hgfs/shared```

if no files are listed of the folder is not found run this:

- ```sudo vmhgfs-fuse .host:/ /mnt/hgfs/ -o allow_other -o uid=1000```

otherwise continue with this step if the files exist.

- ```sudo sh /mnt/hgfs/shared/fedora-config.sh```

fedora-config.sh will run the following:

>```
>sudo cp /mnt/hgfs/shared/i8_namespace.cnd /opt/fcrepo/config/i8_namespace.cnd
>sudo chown tomcat:tomcat /opt/fcrepo/config/i8_namespace.cnd
>sudo chmod 644 /opt/fcrepo/config/i8_namespace.cnd
>sudo touch /opt/fcrepo/config/allowed_hosts.txt
>sudo chown tomcat:tomcat /opt/fcrepo/config/allowed_hosts.txt
>sudo chmod 644 /opt/fcrepo/config/allowed_hosts.txt
>sudo -u tomcat echo "http://localhost:80/" >> /opt/fcrepo/config/allowed_hosts.txt
>sudo cp /mnt/hgfs/shared/repository.json /opt/fcrepo/config/
>sudo chown tomcat:tomcat /opt/fcrepo/config/repository.json
>sudo chmod 644 /opt/fcrepo/config/repository.json
>sudo cp /mnt/hgfs/shared/fcrepo-config.xml /opt/fcrepo/config/
>sudo chmod 644 /opt/fcrepo/config/fcrepo-config.xml
>sudo chown tomcat:tomcat /opt/fcrepo/config/fcrepo-config.xml
>sudo cp /mnt/hgfs/shared/tomcat-users.xml /opt/tomcat/conf/tomcat-users.xml
>sudo chmod 600 /opt/tomcat/conf/tomcat-users.xml
>sudo chown tomcat:tomcat /opt/tomcat/conf/tomcat-users.xml
>```

note: double check /opt/fcrepo/config/allowed_hosts.txt got created

- ```sudo nano /opt/tomcat/bin/setenv.sh```

uncomment line 5, comment line 4 (CTL-c) shows line number
save (CTL-o) exit (CTL+x)

visit: https://github.com/fcrepo/fcrepo/releases

- ```sudo wget -O fcrepo.war https://github.com/fcrepo/fcrepo/releases/download/fcrepo-6.0.0/fcrepo-webapp-6.0.0.war```
- ```sudo mv fcrepo.war /opt/tomcat/webapps```
- ```sudo chown tomcat:tomcat /opt/tomcat/webapps/fcrepo.war```

- ```sudo systemctl restart tomcat```


check here for link...
- https://github.com/Islandora/Syn/releases/
- copy the link (if changed from syn-1.1.1) and replace the link in the command below:

- ```sudo wget -P /opt/tomcat/lib https://github.com/Islandora/Syn/releases/download/v1.1.1/islandora-syn-1.1.1-all.jar```

run Ensure the library has the correct permissions.

- ```sudo sh syn-config.sh```

which runs this series of commands:

>```
>sudo chown -R tomcat:tomcat /opt/tomcat/lib
>sudo chmod -R 640 /opt/tomcat/lib
>sudo mkdir /opt/keys
>sudo openssl genrsa -out "/opt/keys/syn_private.key" 2048
>sudo openssl rsa -pubout -in "/opt/keys/syn_private.key" -out "/opt/keys/syn_public.key"
>sudo chown www-data:www-data /opt/keys/syn*
>sudo cp /mnt/hgfs/shared/syn-settings.xml /opt/fcrepo/config/
>sudo chown tomcat:tomcat /opt/fcrepo/config/syn-settings.xml
>sudo chmod 600 /opt/fcrepo/config/syn-settings.xml
>```
 
edit the context.xml file:

- ```sudo nano /opt/tomcat/conf/context.xml```

Add this line  before the closing </Context> tag:
>```
>    <Valve className="ca.islandora.syn.valve.SynValve" pathname="/opt/fcrepo/config/syn-settings.xml"/>
></Context>
>```

(note above for spelling errors: valve V A L V E not Value)

- ```sudo systemctl restart tomcat```

#### blazegraph

- ```sudo mkdir -p /opt/blazegraph/data```
- ```sudo mkdir /opt/blazegraph/conf```
- ```sudo chown -R tomcat:tomcat /opt/blazegraph```

- ```cd /opt```
- ```sudo wget -O blazegraph.war https://repo1.maven.org/maven2/com/blazegraph/bigdata-war/2.1.5/bigdata-war-2.1.5.war```
- ```sudo mv blazegraph.war /opt/tomcat/webapps```
- ```sudo chown tomcat:tomcat /opt/tomcat/webapps/blazegraph.war```

- ```sh /mnt/hgfs/shared/blazegraph_conf.sh```

Typing out config files by hand is not worth the time. this script simplifies the process.

blazegraph_conf.sh 
configure logging
RWStore.properties
blazegraph.config
inference.nt

>```
>sudo cp /mnt/hgfs/shared/log4j.properties /opt/blazegraph/conf/
>sudo chown tomcat:tomcat /opt/blazegraph/conf/log4j.properties
>sudo chmod 644 /opt/blazegraph/conf/log4j.properties
>sudo cp /mnt/hgfs/shared/RWStore.properties /opt/blazegraph/conf
>sudo cp /mnt/hgfs/shared/blazegraph.properties /opt/blazegraph/conf
>sudo cp /mnt/hgfs/shared/inference.nt /opt/blazegraph/conf
>sudo chown tomcat:tomcat -R /opt/blazegraph/conf
>sudo chmod -R 644 /opt/blazegraph/conf
>```

- ```sudo nano /opt/tomcat/bin/setenv.sh```

comment out line 5 uncomment line 6

save (CTL+o) quit (CTL+x)

- ```sudo systemctl restart tomcat```

- ```sudo curl -X POST -H "Content-Type: text/plain" --data-binary @/opt/blazegraph/conf/blazegraph.properties http://localhost:8080/blazegraph/namespace```

If this worked correctly, Blazegraph should respond with "CREATED: islandora" to let us know it created the islandora namespace.

(sudo?)
- ```curl -X POST -H "Content-Type: text/plain" --data-binary @/opt/blazegraph/conf/inference.nt http://localhost:8080/blazegraph/namespace/islandora/sparql```

If this worked correctly, Blazegraph should respond with some XML letting us
know it added the 2 entries from inference.nt to the namespace.



## solr

- ``` sudo wget https://dlcdn.apache.org/lucene/solr/8.11.2/solr-8.11.2.tgz ```
- ```sudo tar -xzvf solr-8.11.2.tgz```
- ```sudo solr-8.11.2/bin/install_solr_service.sh solr-8.11.2.tgz```

increase filesize (optional)

- ```sudo su```
- ```sudo echo "fs.file-max = 45535" >> /etc/sysctl.conf```
- ```sudo sysctl -p```

(CTL + D) to exit root.

create solr core

- ```cd /opt/solr```
- ```sudo mkdir -p /var/solr/data/islandora8/conf```
- ```sudo cp -r example/files/conf/* /var/solr/data/islandora8/conf```
- ```sudo chown -R solr:solr /var/solr```
- ```sudo -u solr bin/solr create -c islandora8 -p 8983```

A warning will print:

warning using _default configset with data driven scheme functionality. NOT RECCOMENDED for production use. To turn off: bin/solr/ config -c islandora8 -p 8983 -action set-user-property -property update.autoCreateFields -value false

should also say:
"Created new core 'islandora8'

#### drupal search api

- ```cd /opt/drupal```
- ```sudo -u www-data composer require drupal/search_api_solr:^4.2```
- ```drush -y en search_api_solr```

#change vm network adapter to Host-Only (right click vm, settings, etc, save)
#back in the vm 

- ```ip addr show```

note the second ip
in my case it was XXX.XXX.XXX.XXX (don't put your ip in github...)

configure in GUI by visiting the ip above XXX.XXX.XXX.XXX:80 in browser (firefox)
log in with islandora:islandora

you may need to restart apache:

- ```sudo systemctl restart apache2```




navigate to : XXX.XXX.XXX.XXX:80/user/login

then: XXX.XXX.XXX.XXX:80/admin/config/search/search-api
or localhost:80/admin/config/search/search-api

- click add server


these are the options: see documentation for help:
- https://islandora.github.io/documentation/installation/manual/installing_solr/

**config settings in gui**

Server name: islandora8

Enabled: X

backend: Solr 

Standard X

solr core :islandora8 

click advanced config:

    solr.install.dir: /opt/solr


click Save


NOTICE You can ignore the error about an incompatible Solr schema; we're going to set this up in the next step. 

apply solr configs
- click back into the vm

- ```cd /opt/drupal```
- ```drush solr-gsc islandora8 /opt/drupal/solrconfig.zip```
- ```unzip -d ~/solrconfig solrconfig.zip```
- ```sudo cp ~/solrconfig/* /var/solr/data/islandora8/conf```
- ```sudo systemctl restart solr```

#adding an index
#In gui navigate XXX.XXX.XXX.XXX or localhost:80/admin/config/search/search-api/add-index
 
 **configure index via gui***

Index name: Islandora 8 Index

Content: X

File: X

Server: islandora8

Enabled X

click Save


#### crayfish microservices

click back into vm
change vm back to bridged network (only if needed) (right click vm, settings, network adaptor, save)

- ```ping www.google.com```
- do this a few times until you get bytes back

- ```sh /mnt/hgfs/shared/crayfish_reqs.sh```

let this execute: takes a while... maybe take a hydration break. It's running the following:

>```
>sudo add-apt-repository -y ppa:lyrasis/imagemagick-jp2
>sudo apt update
>sudo apt -y install imagemagick tesseract-ocr ffmpeg poppler-utils
>cd /opt
>sudo git clone https://github.com/Islandora/Crayfish.git crayfish
>sudo chown -R www-data:www-data crayfish
>sudo -u www-data composer install -d crayfish/Homarus
>sudo -u www-data composer install -d crayfish/Houdini
>sudo -u www-data composer install -d crayfish/Hypercube
>sudo -u www-data composer install -d crayfish/Milliner
>sudo -u www-data composer install -d crayfish/Recast
>sudo mkdir /var/log/islandora
>sudo chown www-data:www-data /var/log/islandora
>```

# moving config files over

(not microservices-conf.sh )

- ```sh /mnt/hgfs/shared/microservices-config.sh```

runs the following, so very tedious, no one should ever want to type all this out, but this makes certain all the files are named correctly and have the correct permissions, it also beats typing out these configs.
>```
>#!/bin/bash
>sudo cp /mnt/hgfs/shared/homarus.config.yaml /opt/crayfish/Homarus/cfg/config.yaml
>sudo cp /mnt/hgfs/shared/houdini.services.yaml /opt/crayfish/Houdini/config/services.yaml
>sudo cp /mnt/hgfs/shared/crayfish_commons.yml /opt/crayfish/Houdini/config/packages/crayfish_commons.yml
>sudo cp /mnt/hgfs/shared/monolog.yml /opt/crayfish/Houdini/config/packages/monolog.yml
>sudo cp /mnt/hgfs/shared/security.yml /opt/crayfish/Houdini/config/packages/security.yml
>sudo cp /mnt/hgfs/shared/hypercube.config.yaml /opt/crayfish/Hypercube/cfg/config.yaml 
>sudo cp /mnt/hgfs/shared/milliner.config.yaml /opt/crayfish/Milliner/cfg/config.yaml
>sudo cp /mnt/hgfs/shared/recast.config.yaml /opt/crayfish/Recast/cfg/config.yaml
>sudo chown www-data:www-data /opt/crayfish/Homarus/cfg/config.yaml
>sudo chmod 644 /opt/crayfish/Homarus/cfg/config.yaml
>sudo chown www-data:www-data /opt/crayfish/Houdini/config/services.yaml
>sudo chmod 644 /opt/crayfish/Houdini/config/services.yaml
>sudo chown www-data:www-data /opt/crayfish/Houdini/config/packages/crayfish_commons.yml
>sudo chmod 644 /opt/crayfigsh/Houdini/config/packages/crayfish_commons.yml
>sudo chown www-data:www-data /opt/crayfish/Houdini/config/packages/monolog.yml
>sudo chmod 644 /opt/crayfish/Houdini/config/packages/monolog.yml
>sudo chown www-data:www-data /opt/crayfish/Houdini/config/packages/security.yml
>sudo chmod 644 /opt/crayfish/Houdini/config/packages/security.yml
>sudo chown www-data:www-data /opt/crayfish/Hypercube/cfg/config.yaml 
>sudo chmod 644 /opt/crayfish/Hypercube/cfg/config.yaml 
>sudo chown www-data:www-data /opt/crayfish/Milliner/cfg/config.yaml
>sudo chmod 644 /opt/crayfish/Milliner/cfg/config.yaml
>sudo chown www-data:www-data /opt/crayfish/Recast/cfg/config.yaml
>sudo chmod 644 /opt/crayfish/Recast/cfg/config.yaml
>```

configure apache confs for microservices
- ```sh /mnt/hgfs/shared/microservices-conf.sh```

microservices-conf.sh will be copying a lot of config files from the shared folder

>```
>#!/bin/bash
>sudo cp /mnt/hgfs/shared/Houdini.conf /etc/apache2/conf-available/Houdini.conf
>sudo chown root:root /etc/apache2/conf-available/Houdini.conf 
>sudo chmod 644 /etc/apache2/conf-available/Houdini.conf
>sudo cp /mnt/hgfs/shared/Homarus.conf /etc/apache2/conf-available/Homarus.conf 
>sudo chown root:root /etc/apache2/conf-available/Homarus.conf 
>sudo chmod 644 /etc/apache2/conf-available/Homarus.conf 
>sudo cp /mnt/hgfs/shared/Hypercube.conf /etc/apache2/conf-available/Hypercube.conf 
>sudo chown root:root /etc/apache2/conf-available/Hypercube.conf 
>sudo chmod 644 /etc/apache2/conf-available/Hypercube.conf 
>sudo cp /mnt/hgfs/shared/Milliner.conf /etc/apache2/conf-available/Milliner.conf
>sudo chown root:root /etc/apache2/conf-available/Milliner.conf
>sudo chmod 644 /etc/apache2/conf-available/Milliner.conf
>sudo cp /mnt/hgfs/shared/Recast.conf /etc/apache2/conf-available/Recast.conf 
>sudo chown root:root /etc/apache2/conf-available/Recast.conf 
>sudo chmod 644 /etc/apache2/conf-available/Recast.conf 
>```

Installing Karaf and Alpaca

this gives us some but not all requirements.

- ```sudo apt install -y activemq```
- ```cd /opt```
- ```sudo wget  http://archive.apache.org/dist/activemq/5.15.11/apache-activemq-5.15.11-bin.tar.gz```
- ```sudo tar -xvzf apache-activemq-5.15.11-bin.tar.gz```
- ```sudo mv apache-activemq-5.15.11 activemq```
- ```sudo chown -R activemq:activemq /opt/activemq```

- ```sudo nano /etc/systemd/system/activemq.service```

Edit the file like so:
>```
>[Unit]
>Description=Apache ActiveMQ
>After=network.target
>[Service]
>Type=forking
>User=activemq
>Group=activemq
>
>ExecStart=/opt/activemq/bin/activemq start
>ExecStop=/opt/activemq/bin/activemq stop
>
>[Install]
>WantedBy=multi-user.target
>```

- ```sudo systemctl daemon-reload```
- ```sudo systemctl start activemq```
- ```sudo systemctl enable activemq```


---
lacking some documentation here:
- https://websiteforstudents.com/how-to-install-apache-activemq-on-ubuntu-20-04-18-04/





#check version: Installed:  5.15.11-1
- ```sudo apt-cache policy activemq```

- ```sudo addgroup karaf```
- ```sudo adduser karaf --ingroup karaf --home /opt/karaf --shell /usr/bin```

(made password karaf)
enter on all prompts
press y

get karaf : latest 4.3.3 recc (could try 43)
KARAF_TARBALL_LINK: It’s recommended to get the most recent version of Karaf 4.3.3 This will depend on the current version of Karaf, which can be found on the Karaf downloads page under “Karaf Runtime”. Like Solr, you can’t directly wget these links, but clicking on the .tar.gz link for the binary distribution will bring you to a list of mirrors, as well as provide you with a recommended mirror you can use here.

visit https://karaf.apache.org/download.html : http://www.apache.org/dyn/closer.lua/karaf/4.3.3/apache-karaf-4.3.3.tar.gz
then get : https://dlcdn.apache.org/karaf/4.3.3/apache-karaf-4.3.3.tar.gz

- ```cd /opt```
- ```sudo wget -O karaf.tar.gz https://dlcdn.apache.org/karaf/4.4.1/apache-karaf-4.4.1-src.tar.gz```
- ```sudo tar -xzvf karaf.tar.gz```
- ```sudo chown -R karaf:karaf apache-karaf-4.4.1```
- ```sudo mv apache-karaf-4.4.1/* /opt/karaf```

- ```sudo mkdir /var/log/karaf```
- ```sudo chown karaf:karaf /var/log/karaf```




- ```sudo cp /mnt/hgfs/shared/org.pos4j.pax.logging.cfg /opt/karaf/etc/org.pos4j.pax.logging.cfg```
- ```sudo chown karaf:karaf /opt/karaf/etc/org.pos4j.pax.logging.cfg```
- ```sudo chmod 644 /opt/karaf/etc/org.pos4j.pax.logging.cfg```

- ```sudo su```

**NO SUCH FILE OR DIRECTORY { **
- ```sudo echo '#!/bin/sh \nexport JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"' >> /opt/karaf/bin/setenv```

(CTL-d) #to exit from root account

- ```sudo chown karaf:karaf /opt/karaf/bin/setenv```
- ```sudo chmod 755 /opt/karaf/bin/setenv```

- ```sudo nano /opt/karaf/etc/users.properties```

#uncomment lines:
Before:

    32 | # karaf = karaf,_g_:admingroup

    33 | # _g_\:admingroup = group,admin,manager,viewer,systembundles,ssh

After:

    32 | karaf = karaf,_g_:admingroup

    33 | _g_\:admingroup = group,admin,manager,viewer,systembundles,ssh


- ```sudo -u karaf /opt/karaf/bin/start```

You may want to wait a bit for Karaf to start.
If you're not sure whether or not it's running, you can always run:

- ```ps aux | grep karaf```


#run these:
- ```/opt/karaf/bin/client feature:install wrapper```
- ```/opt/karaf/bin/client wrapper:install```
- ```/opt/karaf/bin/stop```

- ```sudo systemctl enable /opt/karaf/bin/karaf.service```
- ```sudo systemctl start karaf```
- ```sudo systemctl status karaf```
- ```q```
- (to quit type q)
- 
**STILL NO FILE EXISTS }**

### Alpaca

check versions of things used in current ansible build or something. the documentation is off from last update, many new versions avail...

from before:
ACTIVEMQ_KARAF_VERSION  5.15.11 

stick with 2.X.X

visit https://mvnrepository.com/artifact/org.apache.camel.karaf/apache-camel The latest version of Apache Camel 2.x.x

APACHE_CAMEL_VERSION https://repo1.maven.org/maven2/org/apache/camel/karaf/apache-camel/2.25.4/apache-camel-
2.25.4

visit https://mvnrepository.com/artifact/ca.islandora.alpaca/islandora-karaf
ISLANDORA_KARAF_VERSION 
1.0.5

JENA_OSGI_VERSION
The latest version of the Apache Jena 3.x OSGi features
3.17.0

(note /xml/ not .xml)


- ```/opt/karaf/bin/client repo-add mvn:org.apache.activemq/activemq-karaf/5.15.11/xml/features```
- ```/opt/karaf/bin/client repo-add mvn:org.apache.camel.karaf/apache-camel/2.25.4/xml/features```
- ```/opt/karaf/bin/client repo-add mvn:ca.islandora.alpaca/islandora-karaf/1.0.5/xml/features```

(This shouldn't be strictly necessary, but appears to be a missing) upstream dependency for some fcrepo features

- ```/opt/karaf/bin/client repo-add mvn:org.apache.jena/jena-osgi-features/3.17.0/xml/features```


- ```sudo -u karaf nano /opt/karaf/etc/ca.islandora.alpaca.http.client.cfg```

edit the file to containt this text:

>```
>token.value=islandora
>```

- ```sudo chmod 644 /opt/karaf/etc/ca.islandora.alpaca.http.client.cfg```

- ```sh /mnt/hgfs/shared/karaf-config.sh```

executes the following file copies:

>```
>!/bin/bash
>sudo cp /mnt/hgfs/shared/org.fcrepo.camel.indexing.triplestore.cfg /opt/karaf/etc/org.fcrepo.camel.indexing.triplestore.cfg
>sudo chown karaf:karaf /opt/karaf/etc/org.fcrepo.camel.indexing.triplestore.cfg
>sudo chmod 644 /opt/karaf/etc/org.fcrepo.camel.indexing.triplestore.cfg
>sudo cp /mnt/hgfs/shared/ca.islandora.alpaca.indexing.triplestore.cfg /opt/karaf/etc/ca.islandora.alpaca.indexing.triplestore.cfg 
>sudo chown karaf:karaf /opt/karaf/etc/ca.islandora.alpaca.indexing.triplestore.cfg 
>sudo chmod 644 /opt/karaf/etc/ca.islandora.alpaca.indexing.triplestore.cfg 
>sudo cp /mnt/hgfs/shared/ca.islandora.alpaca.indexing.fcrepo.cfg /opt/karaf/etc/ca.islandora.alpaca.indexing.fcrepo.cfg
>sudo chown karaf:karaf /opt/karaf/etc/ca.islandora.alpaca.indexing.fcrepo.cfg
>sudo chmod 644 /opt/karaf/etc/ca.islandora.alpaca.indexing.fcrepo.cfg
>```

- ```sh /mnt/hgfs/shared/karaf-blueprints.sh```

 more configuration via file copying and permissions

>```
>#!/bin/bash
>sudo cp /mnt/hgfs/shared/ca.islandora.alpaca.connector.ocr.blueprint.xml /opt/karaf/deploy/ca.islandora.alpaca.connector.ocr.blueprint.xml
>sudo chown karaf:karaf /opt/karaf/deploy/ca.islandora.alpaca.connector.ocr.blueprint.xml
>sudo chmod 644 /opt/karaf/deploy/ca.islandora.alpaca.connector.ocr.blueprint.xml
>sudo cp /mnt/hgfs/shared/ca.islandora.alpaca.connector.houdini.blueprint.xml /opt/karaf/deploy/ca.islandora.alpaca.connector.houdini.blueprint.xml
>sudo chown karaf:karaf /opt/karaf/deploy/ca.islandora.alpaca.connector.houdini.blueprint.xml
>sudo chmod 644 /opt/karaf/deploy/ca.islandora.alpaca.connector.houdini.blueprint.xml
>sudo cp /mnt/hgfs/shared/ca.islandora.alpaca.connector.homarus.blueprint.xml /opt/karaf/deploy>/ca.islandora.alpaca.connector.homarus.blueprint.xml
>sudo chown karaf:karaf /opt/karaf/deploy/ca.islandora.alpaca.connector.homarus.blueprint.xml
>sudo chmod 644 /opt/karaf/deploy/ca.islandora.alpaca.connector.homarus.blueprint.xml
>sudo cp /mnt/hgfs/shared/ca.islandora.alpaca.connector.fits.blueprint.xml /opt/karaf/deploy/ca.islandora.alpaca.connector.fits.blueprint.xml
>sudo chown karaf:karaf /opt/karaf/deploy/ca.islandora.alpaca.connector.fits.blueprint.xml
>sudo chmod 644 /opt/karaf/deploy/ca.islandora.alpaca.connector.fits.blueprint.xml
>```

- ```sh /mnt/hgfs/shared/karaf-features.sh```

contains:

>```
>/opt/karaf/bin/client feature:install camel-blueprint
>/opt/karaf/bin/client feature:install activemq-blueprint
>/opt/karaf/bin/client feature:install fcrepo-service-activemq
># This again should not be strictly necessary, since this isn't the triplestore
># we're using, but is being included here to resolve the aforementioned
># missing link in the dependency chain.
>/opt/karaf/bin/client feature:install jena
>/opt/karaf/bin/client feature:install fcrepo-camel
>/opt/karaf/bin/client feature:install fcrepo-indexing-triplestore
>/opt/karaf/bin/client feature:install islandora-http-client
>/opt/karaf/bin/client feature:install islandora-indexing-triplestore
>/opt/karaf/bin/client feature:install islandora-indexing-fcrepo
>/opt/karaf/bin/client feature:install islandora-connector-derivative
>```

#configure drupal

#settings.php

- ```sudo nano /opt/drupal/web/sites/default/settings.php```

add the following to the end of the file:

>```
>$settings['trusted_host_patterns'] = [
>  'localhost',
>  '^192\.168\.12\.131$',
>];
>$settings['flysystem'] = [
> 'fedora' => [
> 'driver' => 'fedora',
> 'config' => [
>  'root' => 'http://localhost:8080/fcrepo/rest/',
> ],
>],
>];
>```

- ```cd /opt/drupal```
- ```drush -y cr```


- ```sudo sh /mnt/hgfs/shared/islandora_install.sh```

The above script will run the following (to save you time typing version numbers...)

>```
>#!/bin/bash
>cd /opt/drupal
># This is a convenience piece that will help speed up most of the rest of our
># process working with Composer and Drupal.
>sudo -u www-data composer require zaporylie/composer-drupal-optimizations:^1.0
># Since islandora_defaults is near the bottom of the dependency chain, requiring
># it will get most of the modules and libraries we need to deploy a standard
># Islandora site.
>sudo -u www-data composer require islandora/islandora_defaults:dev-8.x-1.x
># These can be considered important or required depending on your site's
># requirements; some of them represent dependencies of Islandora submodules.
>sudo -u www-data composer require drupal/pdf:1.x-dev
>sudo -u www-data composer require drupal/rest_oai_pmh:^1.0
>sudo -u www-data composer require drupal/facets:^1.3
>sudo -u www-data composer require drupal/restui:^1.16
>sudo -u www-data composer require drupal/rdfui:^1.0-beta1
>sudo -u www-data composer require drupal/content_browser:^1.0@alpha
># These tend to be good to enable for a development environment, or just for a
># higher quality of life when managing Islandora. That being said, devel should
># NEVER be enabled on a production environment, as it intentionally gives the
># user tools that compromise the security of a site.
>sudo -u www-data composer require drupal/console:~1.0
>sudo -u www-data composer require drupal/devel:^2.0
>sudo -u www-data composer require drupal/admin_toolbar:^2.0
># Islandora also provides a theme called Carapace designed to work well out of
># the box with an Islandora site.
>sudo -u www-data composer require islandora/carapace:dev-8.x-3.x
>```

something happened, no no, did one fail?
something about sudo -u www-data composer require zaporylie/composer-drupal-optimizations:^1.0
no 
yes

- ```sh /mnt/hgfs/shared/isla_lib.sh```

The script above will execute the following (save on typing long links...)

>```
>#!/bin/bash
>#add these before enable
>composer require drupal/masonry
>composer require drupal/libraries
>cd web
>mkdir libraries
>cd libraries
>mkdir masonry/dist
>mkdir imagesloaded
>wget -O masonry/dist/masonry.pkgd.min.js https://unpkg.com/masonry-layout@4.2.2/dist/masonry.pkgd.min.js
>wget -O imagesloaded/imagesloaded.pkgd.min.js https://unpkg.com/imagesloaded@4.1.4/imagesloaded.pkgd.min.js
>```

- ```sh /mnt/hgfs/shared/islandora_en.sh```

set vm back into host only network 

if provided host name is not valid... try editing /setting.php change ip to match

set vm back to host only network setting

Adding a JWT Configuration to Drupal

To allow our installation to talk to other services via Syn, we need to establish a Drupal-side JWT configuration using the keys we generated at that time.

Log onto your site as an administrator at /user, then navigate to /admin/config/system/keys/add. Some of the settings here are unimportant, but pay close attention to the Key type, which should match the key we created earlier (an RSA key), and the File location, which should be the ultimate location of the key we created for Syn on the filesystem, /opt/keys/syn_private.key.

Adding a JWT RSA Key

Click Save to create the key.

Once this key is created, navigate to /admin/config/system/jwt to select the key you just created from the list. Note that before the key will show up in the Private Key list, you need to select that key's type in the Algorithm section, namely RSASSA-PKCS1-v1_5 using SHA-256 (RS256).

Configuring the JWT RSA Key for Use

See instructions:
- https://islandora.github.io/documentation/installation/manual/configuring_drupal/#adding-a-jwt-configuration-to-drupal

visit http://XXX.XXX.XXX.XXX/admin/config/system/jwt


follow drupal config instructions:
- https://islandora.github.io/documentation/installation/manual/configuring_drupal/#islandora

config canaloupe
- https://islandora.github.io/documentation/installation/manual/configuring_drupal/#configuring-islandora-iiif

cantaloupe endpoint was not 8080
this worked for cantaloupe config:

had to unzip cantaloupe missed the unzip for some reason
- http://XXX.XXX.XXX.XXX:8182/iiif/2

Nav to openseadragog
- http://XXX.XXX.XXX.XXX:8182/admin/config/media/openseadragon

add to IIIf Image server location: http://XXX.XXX.XXX.XXX:8182/iiif/2
select IIIF Manifest from dropdown
save

navigate to flysystem settings
http://XXX.XXX.XXX.XXX:8182/admin/config/media/file-system
choose the flysystem button and save (scroll down)

give the admin fedoraAdmin role

- ```cd /opt/drupal```
- ```sudo -u www-data drush -y urol "fedoraadmin" islandora```

- ```sudo -u www-data drush -y -l localhots --userid=1 mim --group=islandora```

uninstall 
/admin/modules/uninstall 
search_defaults

reslove a dependency

- ```sudo -u www-data composer require 'drupal/jquery_ui_accordion:^1.1'```
- ```drush en -y jquery_ui_accordion```



