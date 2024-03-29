# Describes adapted build using shared folders and scripts to manually build islandora.

### Notes

This build is adapted from this link and it may help to read it or follow along:
- https://islandora.github.io/documentation/installation/manual/introduction/
- some downloads will need new links as new versions come out
- many config files and scripts are included to eliminate a lot of typing within the build. 
- you'll need to enable shared folders in the vm settings and place the files in the shared folder
- code outside a comment or gui instruction is intended to be executed in the vmware terminal, ie:
- `whoami`
- you should know the linux command line fairly well
- you should know how to use nano or some other command line editor vi/m emacs
- you need to have downloaded vmware, and an ubuntu 20.04 server image.

**Pre-BUILD Requirements**

### download vmware
- https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html
- LSU has a way to get a license: https://software.grok.lsu.edu/Article.aspx?articleId=20512
- visit LSU box and download shared folder contents these will be your configuration files and others needed for the build add them to your shared folder in the vmware machine
- https://lsu.box.com/s/o2uy4yx57lrzlu6tbpmuis6n5ocgq1r9

- create a vmware machine 
- install ubuntu 20.04 Server from an ISO image:
- https://ubuntu.com/download/server#downloads
- 20 GB on one file for disk size 

- make sure the vm is on bridged network setting for installation
- (right click the vm, go to settings>network adapter, select "Bridged", then save)

- go through the os installation process
- all prompts you don't need to change anything unless you want openssh (for remote access via ssh)
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
- ```sudo apt -y upgrade```
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
- ```sudo apt -y install php8.1 php8.1-cli php8.1-common php8.1-curl php8.1-dev php8.1-gd php8.1-imap php8.1-mbstring php8.1-opcache php8.1-xml php8.1-yaml php8.1-zip libapache2-mod-php8.1 php-pgsql php-redis php-xdebug unzip postgresql```

Edit the postgresql.conf file starting at line 688

- ```sudo nano +688 /etc/postgresql/14/main/postgresql.conf```

(if you don't like nano use a different editor vi/vim/emacs )

change line 688 from 
>```
>#bytea_output 'hex'
>```

change to
>```
>bytea_output 'escape'
>```

(CTL+o)(to save)(CTL+x to close)

- ```sudo systemctl restart postgresql```

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

confirm link:

- ```ls -lart /usr/local/bin/drush```

Expected output will link to /opt/drupal/vendor/drush/drush/drush


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

as of 03-03-23 it was:

https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.73/bin/apache-tomcat-9.0.73.tar.gz


watch for version change
- ``` cd /opt```
- ```sudo wget -O tomcat.tar.gz TOMCAT_TARBALL_LINK```
- ```ls /opt``` to find name of TOMCAT_DIRECTORY: (recently it was: apache-tomcat-9.0.73)
- ```sudo tar -zxvf tomcat.tar.gz```
- ```sudo mv /opt/TOMCAT_DIRECTORY/* /opt/tomcat```
- ```sudo chown -R tomcat:tomcat /opt/tomcat ```

- ```sh /mnt/hgfs/shared/scratch_5.sh```

scratch_5.sh will run the following file:
>```
>#!/bin/bash
>cd /opt
>#O not 0
>sudo wget -O tomcat.tar.gz https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.73/bin/apache-tomcat-9.0.73.tar.gz
>sudo tar -zxvf tomcat.tar.gz
>#don't miss the star*
>sudo mv /opt/apache-tomcat-9.0.73/* /opt/tomcat
>sudo chown -R tomcat:tomcat /opt/tomcat
>```

- ```sh /mnt/hgfs/shared/scratch_6.sh```
l
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

=s======================================

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
s
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
>ssudo -u tomcat echo "http://localhost:80/" >> /opt/fcrepo/config/allowed_hosts.txt
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


copy setenv.sh from /mnt/hgfs/shared/ to /opt/tomcat/bin/

- ```sudo nano /opt/tomcat/bin/setenv.sh```

uncomment line 5, comment line 4 (CTL-c) shows line number
save (CTL-o) exit (CTL+x)

visit: https://github.com/fcrepo/fcrepo/releases

- ```sudo wget -O fcrepo.war https://github.com/fcrepo/fcrepo/releases/download/fcrepo-6.3.0/fcrepo-webapp-6.3.0.war```
- ```sudo mv fcrepo.war /opt/tomcat/webapps```
- ```sudo chown tomcat:tomcat /opt/tomcat/webapps/fcrepo.war```
- ```sudo systemctl restart tomcat```


check here for link...
- https://github.com/Islandora/Syn/releases/
- copy the link (if changed from syn-1.1.1) and replace the link in the command below:

- ```sudo wget -P /opt/tomcat/lib https://github.com/Islandora/Syn/releases/download/v1.1.1/islandora-syn-1.1.1-all.jar```

confirm the library has the correct permissions.

- ```sudo sh /mnt/hgfs/shared/syn-config.sh```

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
su>```

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

#not sure

If this worked correctly, Blazegraph should respond with some XML letting us
know it added the 2 entries from inference.nt to the namespace.


## solr

- ``` sudo wget https://dlcdn.apache.org/lucene/solr/8.11.2/solr-8.11.2.tgz ```
- ```sudo tar -xzvf solr-8.11.2.tgz```
- ```sudo solr-8.11.2/bin/install_solr_service.sh solr-8.11.2.tgz```


#q to quit...

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

#had to cd into /opt/solr/exmaple/files/conf 
#ran
#sudo cp -r . /var/solr/data/islandora8/conf


A warning will print:

warning using _default configset with data driven scheme functionality. NOT RECOMMENDED for production use. To turn off: bin/solr/ config -c islandora8 -p 8983 -action set-user-property -property update.autoCreateFields -value false

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

##encountered some trouble with solr on unzip step


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
>sudo chmod 644 /opt/crayfish/Hypercube/cfg/configl.yaml 
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

enable microservices

- ```sudo a2enconf Homarus Houdini Hypercube Milliner Recast```
- ```sudo systemctl restart apache2```


### ActiveMQ Karaf Alpaca  section removed. not required for build.

### configure drupal

edit settings.php

- ```sudo nano /opt/drupal/web/sites/default/settings.php```

add the following to the end of the file:

>```
>$settings['trusted_host_patterns'] = [
>  'localhost',
>  'your-site-ip'
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
># sudo -u www-data composer require drupal/devel:^2.0 --with-all-dependencies #not working or conflict with symphony/var-dumper
>sudo -u www-data composer require drupal/admin_toolbar:^2.0
># Islandora also provides a theme called Carapace designed to work well out of
># the box with an Islandora site.
>sudo -u www-data composer require islandora/carapace:dev-8.x-3.x
>```

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

The above script runs:

>```
>cd /opt/drupal
>drush -y en rdf responsive_image syslog serialization basic_auth rest restui search_api_solr facets content_browser pdf admin_toolbar islandora_defaults controlled_access_terms_defaults islandora_breadcrumbs islandora_iiif islandora_oaipmh
>drush -y theme:enable carapace
> drush -y config-set system.theme default carapace
>drush -y cr
>```


Adding a JWT Configuration to Drupal

To allow our installation to talk to other services via Syn, we need to establish a Drupal-side JWT configuration using the keys we generated at that time.

Log onto your site as an administrator at /user, then navigate to /admin/config/system/keys/add. Some of the settings here are unimportant, but pay close attention to the Key type, which should match the key we created earlier (an RSA key), and the File location, which should be the ultimate location of the key we created for Syn on the filesystem, /opt/keys/syn_private.key.

Adding a JWT RSA Key

Click Save to create the key.

Once this key is created, navigate to /admin/config/system/jwt to select the key you just created from the list. Note that before the key shows up in the Private Key list, you need to select that key's type in the Algorithm section, namely RSASSA-PKCS1-v1_5 using SHA-256 (RS256).

Configuring the JWT RSA Key for Use

See instructions:
- https://islandora.github.io/documentation/installation/manual/configuring_drupal/#adding-a-jwt-configuration-to-drupal

visit http://[your-site-ip-address]/admin/config/system/jwt


follow drupal config instructions:
- https://islandora.github.io/documentation/installation/manual/configuring_drupal/#islandora

config canaloupe
- https://islandora.github.io/documentation/installation/manual/configuring_drupal/#configuring-islandora-iiif

cantaloupe endpoint was not 8080
this worked for cantaloupe config:

had to unzip cantaloupe missed the unzip for some reason
- http://[your-site-ip-address]:8182/iiif/2

Nav to openseadragon
- http://[your-site-ip-address]:8182/admin/config/media/openseadragon

add to IIIf Image server location: http://[your-site-ip-address]:8182/iiif/2
select IIIF Manifest from dropdown
save

navigate to flysystem settings
http://[your-site-ip-address]:8182/admin/config/media/file-system
choose the flysystem button and save (scroll down)

give the admin fedoraAdmin role

- ```cd /opt/drupal```
- ```sudo -u www-data drush -y urol "fedoraadmin" islandora```

<!--
this line isn't right
- ```sudo -u www-data drush -y -l localhost --userid=1 mim --group=islandora```
-->

- ```sudo -u www-data drush -y -l localhost --userid=1 mim```

uninstall 
/admin/modules/uninstall 
search_defaults



--skip this dependency isn't needed 
resolve a dependency

- ```sudo -u www-data composer require 'drupal/jquery_ui_accordion:^1.1'```
- ```drush en -y jquery_ui_accordion```


###Theme trouble try remove require install

- ```sudo -u www-data composer remove islandora/carapace```

- ```sudo -u www-data composer require islandora/carapace```

- visit "localhost/admin/appearance" and install a the carapace theme

###require islandora workbench

https://github.com/mjordan/islandora_workbench_integration

- ```cd /opt/drupal```
- ```sudo -u www-data composer require mjordan/islandora_workbench_integration "dev-main"```
- ```drush en -y islandora_workbench_integration```

#### enable rest endpoints for workbench then rebuild the cache

- ```drush cim -y --partial --source=modules/contrib/islandora_workbench_integration/config/optional```

- ```drush cr -y```

- an follow documentation for islandora workbench 

- outside of your virtual machine clone

- git clone https://github.com/mjordan/islandora_workbench_integration

- edit a config.yml file with the http://your-virtualmachine-ip-address in the config

- config file could look something like this:

>```
>Task: create
>host: "http://your-virtualmachine-ip-addr"
>username: islandora
>password: islandora
>input_csv: path-to-your-input.csv
>allow_adding_terms: True
>allow_missing_files: True
>```


### styles folder problem:

- ```chown -R www-data:www-data /opt/web/sites/default/files/styles```


## upload size and max post size

- ```sudo nano /etc/php/8.1/apache2/php.ini```
- change ```post_max_size = 8M``` to ```post_max_size = 200M```
- change ```upload_max_filesize = 8M``` to ```upload_max_filesize = 200M```
- change  ```max_file_uploads = 200``` to an appropriate number #(confirm this limit)
- adjust filesize to an appropriate size for your uploads
- ```sudo systemctl restart apache2```
