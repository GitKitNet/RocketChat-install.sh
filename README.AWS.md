# AWS

Deploying Rocket.Chat on Amazon Web Services
This guide covers the following:
Hosting Rocket.Chat on an Amazon EC2 instance
Hosting a domain name with Amazon Route 53
Securing your server with a free SSL certificate from Let's Encrypt
Launch an EC2 instance
Log into AWS console, open the , click on "Instances" in the left sidebar and click on "Launch Instance" to set up a new EC2 instance. Now follow the steps below:
In the first step search for "Ubuntu Server 18.04 LTS" with "64-bit (x86)" architecture and click on "Select"
Select an instance type of your choice and click "Next"
Adjust the instance details as needed or keep the defaults. Proceed with "Next"
Adjust the storage size and configuration as needed and click on "Next"
Make sure to add a tag called "Name" and assign a value
Allow "SSH", "HTTP" and "HTTPS" in the security group configuration, proceed with "Review and Launch"
Review your instance configuration and confirm with "Launch"
Choose an existing key pair or create a new one and click on "Launch Instance"
Allocate an Elastic IP
Back in the  dashboard, click on "Elastic IPs" in the left sidebar:
Click on "Allocate New Address"
Select "Amazon's pool of IPv4 addresses" and click on "Allocate"
Click on the newly created IP address and select "Associate Elastic IP address"
Select your instance and click "Associate"
In the details below, copy the "Public DNS". You will need it in the DNS step.
(It should be in a format like this: ec2-18-197-161-168.eu-central-1.compute.amazonaws.com)
Configure DNS w/ AWS Route 53
Open the "Route 53" service dashboard:
Create a new hosted zone by clicking on "Create Hosted Zone":
Enter your domain name and select "Public Hosted Zone" as type, then click on "Create"
Select your newly created zone and click on "Create Record Set"
Enter "www" as subdomain (if desired), select Type "CNAME", enter the Public DNS name from the above step to the value field and click "Create"
Get an SSL certificate from Let's Encrypt
We will use Let's Encrypt to get a free & open-source SSL certificate:
SSH to your instance:
```
 ssh -i <path_to_key_file.pem> ubuntu@<public_ip_address>
```
Note: You may replace with domain name if your DNS has resolved.
Install certbot using apt:
```
 sudo apt update
 sudo apt install certbot
```
Obtain certificate from Let's Encrypt:
```
 sudo certbot certonly --standalone --email <emailaddress@email.com> -d <domain.com> -d <subdomain.domain.com>
```
Note: Second (or more) domain is optional.
Optional step: restrict access using security groups
If you would like to restrict traffic to your instance on AWS, you may now adjust the security groups again. Make sure you allow "TCP/22" from your current location for the SSH connection, as well as "TCP/443" from the location you wish to use to access from.
Configure Nginx web server with TLS/SSL
Install Nginx web server:
```
 sudo apt-get install nginx
```
Backup the default config file for reference:
```
 cd /etc/nginx/sites-available
 sudo mv default default.reference
```
Create a new site configuration for Rocket.Chat:
```
 sudo nano /etc/nginx/sites-available/default
 server {
     listen 443 ssl;
​
     server_name <ABC.DOMAIN.COM>;
​
     ssl_certificate /etc/letsencrypt/live/<ABC.DOMAIN.COM>/fullchain.pem;
     ssl_certificate_key /etc/letsencrypt/live/<ABC.DOMAIN.COM>/privkey.pem;
     ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
     ssl_prefer_server_ciphers on;
     ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
​
     root /usr/share/nginx/html;
     index index.html index.htm;
​
     # Make site accessible from http://localhost/
     server_name localhost;
​
     location / {
         proxy_pass http://localhost:3000/;
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection "upgrade";
         proxy_set_header Host $http_host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header X-Forwarded-Proto http;
         proxy_set_header X-Nginx-Proxy true;
         proxy_redirect off;
     }
 }
​
 server {
     listen 80;
​
     server_name <ABC.DOMAIN.COM>;
​
     return 301 https://$host$request_uri;
 }
```
Make sure to replace ABC.DOMAIN.COM with your domain (it appears 4 times). Make sure to update it in the path to your key files as well:
Test the Nginx configuration to make sure there are no syntax errors:
```
 sudo nginx -t
```
If the syntax test went successful, restart Nginx:
```
 sudo systemctl restart nginx
```
Confirm that it is running properly by opening a web browser and going to your domain name. You will get a page stating "502 Bad Gateway". This is expected, since the Rocket.Chat backend is not yet running. Make sure the SSL connection is working properly by clicking the lock icon next to the address bar, make sure it's valid and issued by "Let's Encrypt Authority X3".
Install Docker & Docker Compose
Install Docker (and any dependencies)
```
 sudo apt-get update
 sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 sudo apt-key fingerprint 0EBFCD88
 # confirm the fingerprint matches "9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88"
 sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Install docker-compose:
```
 sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
 sudo chmod +x /usr/local/bin/docker-compose
```
Set up Docker containers
Create local directories
```
 sudo mkdir -p /opt/docker/rocket.chat/data/runtime/db
 sudo mkdir -p /opt/docker/rocket.chat/data/dump
```
Create the docker-compose.yml file, again make sure to replace ABC.DOMAIN.COM with your actual domain name:
```
 sudo nano /opt/docker/rocket.chat/docker-compose.yml
 version: '2'
​
 services:
   rocketchat:
     image: rocket.chat:latest
     command: >
       bash -c
         "for i in `seq 1 30`; do
           node main.js &&
           s=$$? && break || s=$$?;
           echo \"Tried $$i times. Waiting 5 secs...\";
           sleep 5;
         done; (exit $$s)"
     restart: unless-stopped
     volumes:
       - ./uploads:/app/uploads
     environment:
       - PORT=3000
       - ROOT_URL=https://<ABC.DOMAIN.COM>
       - MONGO_URL=mongodb://mongo:27017/rocketchat
       - MONGO_OPLOG_URL=mongodb://mongo:27017/local
     depends_on:
       - mongo
     ports:
       - 3000:3000
​
   mongo:
     image: mongo:4.0
     restart: unless-stopped
     command: mongod --smallfiles --oplogSize 128 --replSet rs0 --storageEngine=mmapv1
     volumes:
       - ./data/runtime/db:/data/db
       - ./data/dump:/dump
​
   # this container's job is just to run the command to initialize the replica set.
   # it will run the command and remove himself (it will not stay running)
   mongo-init-replica:
     image: mongo:4.0
     command: >
       bash -c
         "for i in `seq 1 30`; do
           mongo mongo/rocketchat --eval \"
             rs.initiate({
               _id: 'rs0',
               members: [ { _id: 0, host: 'localhost:27017' } ]})\" &&
           s=$$? && break || s=$$?;
           echo \"Tried $$i times. Waiting 5 secs...\";
           sleep 5;
         done; (exit $$s)"
     depends_on:
     - mongo
```
Start containers:
```
 cd /opt/docker/rocket.chat
 sudo docker-compose up -d
```
Wait a bit for the replica set to be initialized for MongoDB (about 30-60 seconds) and confirm Rocket.Chat is running properly:
```
 sudo docker-compose logs -f rocketchat
```
Use it

Login to your site at https://ABC.DOMAIN.COM
Note: the first user to login will be an administrator user.
