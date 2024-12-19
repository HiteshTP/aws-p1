# =========================================
# COMMANDS TO RUN IN THE APPLICATION SERVER
# =========================================

# ======================================
# INSTALLING MYSQL IN AMAZON LINUX 2023
# ======================================
# (REF: https://dev.to/aws-builders/installing-mysql-on-amazon-linux-2023-1512)

#!/bin/bash
sudo -su ec2-user
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install mysql-community-client -y
mysql --version

# TO TEST CONNECTION BETWEEN APP-SERVER & DATABASE SERVER
mysql -h <RDS-Endpoint> -u <username> -p <Hit Enter & provide your password>

#===============================
# COPYING CONTENT FROM S3 BUCKET
#===============================
cd /home/ec2-user

# !!! IMP !!!
# MODIFY BELOW CODE WITH YOUR S3 BUCKET NAME
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/app-tier app-tier --recursive
cd app-tier
sudo chown -R ec2-user:ec2-user /home/ec2-user/app-tier
sudo chmod -R 755 /home/ec2-user/app-tier

#===============================
# INSTALLING NODEJS
#===============================
# (REF: https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html)

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
npm install -g pm2
npm install
npm audit fix

#===============================
# STARTING INDEX.JS FILE
#===============================
pm2 start index.js 	#(Start Application with PM2, PM2 is process manager for NodeJS)
pm2 logs            #(To see logs, run Ctrl+C to exit)
pm2 startup 			  #(Set PM2 to Start on Boot)
sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v16.20.2/bin /home/ec2-user/.nvm/versions/node/v16.20.2/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user
pm2 save			      #(Save the current configuration)
curl http://localhost:4000/health #(To do the health check)



---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- WEB SERVER COMMAND ----------------------------------------------------------------------------------------------------------
# =========================================
# COMMANDS TO RUN IN THE WEB SERVER
# =========================================

# =========================================
# COPY WEB-TIER CODE FROM S3
# =========================================
#!/bin/bash
sudo -su ec2-user
cd /home/ec2-user

# !!! IMP !!!
# MODIFY BELOW CODE WITH YOUR S3 BUCKET NAME
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/web-tier web-tier --recursive
sudo chown -R ec2-user:ec2-user /home/ec2-user
sudo chmod -R 755 /home/ec2-user

# =========================================
# INSTALLING NODEJS (FOR USING REACT LIBRARY)
# =========================================
# (REF: https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html)	
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
cd /home/ec2-user/web-tier
npm install
npm audit fix

# =========================================
# BUILDING THE APP FOR PRODUCTION
# =========================================
# Below command is used to build the code which can be served by the webserver (Nginx)
cd /home/ec2-user/web-tier
npm run build

# =========================================
# INSTALLING NGINX (WEBSERVER)
# =========================================
# (REF: https://dev.to/0xfedev/how-to-install-nginx-as-reverse-proxy-and-configure-certbot-on-amazon-linux-2023-2cc9)
# NOTE: Before using the nginx.conf file in below code, ensure to add the Internal-Load-Balancer-DNS in the nginx.conf file & upload it to S3

sudo yum install nginx -y	
cd /etc/nginx
sudo mv nginx.conf nginx-backup.conf

# !!! IMP !!!
# MODIFY BELOW CODE WITH YOUR S3 BUCKET NAME
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/nginx.conf . 
sudo chmod -R 755 /home/ec2-user
sudo service nginx restart
sudo chkconfig nginx on
