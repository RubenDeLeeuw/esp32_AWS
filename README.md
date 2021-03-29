
# esp32MQTTonAWS

### Creating a Thing

We start bij creating a thing. you can do this by going to https://us-west-2.console.aws.amazon.com/console/home?region=us-west-2#. go to IoT core => manege => thing. And create a new thing.  The name  is not important but make sure you remember it.  When your thing is created you create a new certificate for this thing and download the 3 keys you are given, put these in a safe place where you can find them easily back becuase we need them later and if you lose them you need to create a thing again. 

### .ino Code

Once you have done this open the .ino file from this project. next you create a secrets.h file in the same folder according to the example found. now we need the keys we got earlier when we created our thing. Open the keys in a texteditor like notepad and copy them. put them on the right place in the code and once you placed all the keys on the right place in the code you can upload them.

### Creating S3 Bucket

Now were going to make a s3 bucket for our thing. go to s3 => bucket and create here a new bucket. Choose the same server as youre thing.
go back to AWS IoT and in act got ot rules here you have to add a new rule. choose a rule name and change the topic to the topi in youre .ino code. next you need to add an action. select store message in amazon s3 bucket. then choose your bucket from the previous step and as key I choose data/${device_id}_${topic()}/${timestamp()}. the data you send should now appear in your bucket.

# cloud api

now were going to create an EC2 virtual machine on aws to store our data. I used an ubuntu server with the t2 micro tear because this one is the cheapest. You don't have to change much in the first 5 steps but notice in the 6th step you have to add http and https. After completing the last step generate a key, give it a name and **download** it. . now open PuTTYgen choose the key you recently downloaded and load it. Now go back to aws EC2 where you can find the ip address of your virtual machine. Now open up PuTTY select ssh as connection type and put in the IP you just got from AWS. 


### Ubuntu

Before you can open the connection on the left side you see a menu here go into ssh and click on auth here click browse and select the file you got from PuTTYgen. now you should be able to open a connection using the open button.

Now you login with the username ubuntu.

Were going to do some commands to update/download everything and soon creating our first mysql database.

Now start by doing the **sudo apt-get update** and **sudo apt-get upgrade**

When this is finished, do the following commands to get an apache server running and a database.

**sudo apt install apache2**

**sudo apt install mysql-server**

**sudo apt install php libapache2-mod-php php-mysql** 

**sudo chmod 777 /var/www/html** 

**sudo systemctl restart apache2** 

**sudo apt install certbot** 

**sudo apt install python3-certbot-apache**

### Mysql Database

**'sudo mysql'**

now you have entered the mysql command line now follow these steps 

**CREATE DATABASE [DATABASENAME];**

**CREATE USER '[USERNAME]'@'localhost' IDENTIFIED BY '[PASSWORD]';**

**GRANT ALL PRIVILEGES ON [DATABASENAME].* TO '[USERNAME]'@'localhost';**

**FLUSH PRIVILEGES;**

**USE [DATABASENAME];**

**CREATE TABLE esp32 (**

**ID INT NOT NULL AUTO_INCREMENT,**

**Device_id VARCHAR (100) NOT NULL,**

**Temperature Float,**

**Humidity Float,**

**Datetime DATETIME DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (ID)**

**);**

**quit**

Now you have made a database for your data to test if this was done correctly, copy/create the test.php and mysql_connect.php from the cloud_api folder to the /var/www/html directory. In this same directory you create a secrets.php from the example_secrets in the cloud_api.

Now in your web browser you can test if this works by surfing to your ip followed by /test.php.


### http => https

Now to make your connection https instead of http you need to create an account on the noip https://www.noip.com/. Here we create a dns for your ip by clicking the create hostname button. After choosing the best hostname click on it in the no-ip hostname list and putting in your virtual machine ip as the destination. Do take notice that if your machine is closed you will have to do this step gain if you want this domain to keep on working. 

### Certbot

Now to actually get certified we need to use certbot just use the **sudo certbot --apache** command and fill in your information and at the end select option 2 to force everyone to use https.
Now for the finishing touches add the api.php file and list.php files to the var/www/html folder.
To now make aws send your data to the correct location go back to the rule we made in part 1 and add a new action to this rule a http action with as link your v link followed by /api.php. and three headers withe the keys: device_id, temperature and humidity and values: ${devide_id}, ${temperature} and ${humidity} respectively.
Now go to destinations and add the api.php link here aswell and to enable it add the api token found using the sudo tail /var/log/apache2/access.log command.
Now by opening your site followed by list.php you should see your esp32 data.


# visualization

 we want to display our data using grafana.

### influxdb

First install infuxdb as our database.

**sudo curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -**

next add and influx data repository

**source /etc/lsb-release echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list**

now update the package list and install influxdb

**sudo apt update sudo apt install influxdb -y**

Now to launch influxdb on boot start the influxdb service.

**sudo systemctl start influxdb** 

**sudo systemctl enable influxdb**

to see if influxdb is running correctly do:

**netstat -plntu**

if in this list port 8088 and 8086 are in a LISTEN state it is running.

now create a user and database this is about the same as we did in mysql.

you start with the influx command

**influx**

to connect on the 8086 port now you start by creating a database and user for the telegraf client.

**create database telegraf** 

**create user telegraf with password 'hakase-ndlr'**

To make sure these have been created use:

**show databases**

**show users**

Here you will see the database name telegraf and user name telegraf in you list.

telegraf
To install telegraf.

**sudo apt install telegraf -y**

Now again we want telegraf to start on boot so we start the telegraf service.

**sudo systemctl start telegraf**

**sudo systemctl enable telegraf**

To check if telegraf actualy started, execute.

**sudo systemctl status telegraf**


First we make a backup of the telegraf.conf file to restore the original configuration incase of mistakes.

**cd /etc/telegraf/ mv telegraf.conf telegraf.conf.default**

now remove the original telegraf.conf

**rm telegraf.conf**

now create a new telegraf.conf  and past the data found in the telegraf.txt file on the github

**nano telegraf.conf**

now restart telegraf

**sudo systemctl restart telegraf**

And test if it worked by using these 3 commands

**sudo telegraf -test -config /etc/telegraf/telegraf.conf --input-filter cpu**

**sudo telegraf -test -config /etc/telegraf/telegraf.conf --input-filter net**

**sudo telegraf -test -config /etc/telegraf/telegraf.conf --input-filter mem**

### grafana

To install grafana  do

**sudo apt-get install -y apt-transport-https** 

**sudo apt-get install -y software-properties-common wget** 

**wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -**

**echo "deb https://packages.grafana.com/enterprise/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list**

**sudo apt update**

**sudo apt install grafana-enterprise**

**sudo systemctl daemon-reload**

**sudo systemctl start grafana-server**

**sudo systemctl status grafana-server**

**sudo systemctl enable grafana-server.service**

now go back to aws and here we have to open the port 3000 to the world in order to access grafana from other devices. To do this go to ec2 on aws and in instances click on your instance now at the bottom there is a window with the instance details. Here you click on security then on the link under the title security groups. This link will bring you to the launch wizard for your security. Here you click on edit inbound rules and add a custom TCP rule for the port 3000 and with the course 0.0.0.0/0.

Once the port is open you can surf to you ip flowed by the port 3000 to go to the grafana interface here you add mysql as your data source. now when you can create a dashboards with some graphs to see youre data

# lambda

To create a lambda function go to the compute section on aws and select lambda. in functions you click create function. Normally ‘author from scratch’ is selected, if not select this then you can fill in the name of this function and in runtime select python, the version doesn't really matter.

Now you should see an add trigger button click here and select AWS IoT then select custom IoT rule and existing rule. Now you should be able to select the rule we made before to send the data from iot to the bucket and the api on the virtual machine also make sure to select the enable trigger that will make your life easier. Now to actually make the function useful you should add the code found in de lmada folder in the AWS IoT file to the lambda_funtion.py file on aws and editing the led topic to your own and correcting the server to the one you are on now.

