# Guacamole-Setup-Guide-
Instructions on how to setup Apache Guacamole Server with Database and Protocols RDP,VNC & SSH

## Youtube Video Guide
https://youtu.be/v_XuHrTRRpQ

## Step 1 Download the require packages
```
sudo apt install gcc libcairo2-dev libpng-dev libtool-bin uuid-dev libossp-uuid-dev make pkg-config libssh2-1-dev libpango1.0-dev libssl-dev cmake git ninja-build build-essential libkrb5-dev libxi-dev libcups2-dev libfuse3-dev pkg-config libusb-1.0-0-dev mysql-server libvncserver-dev xxd apache2-utils
```
## Step 2 Download the Required files
```
wget https://apache.org/dyn/closer.lua/guacamole/1.6.0/source/guacamole-server-1.6.0.tar.gz?action=download https://apache.org/dyn/closer.lua/guacamole/1.6.0/binary/guacamole-auth-jdbc-1.6.0.tar.gz?action=download https://apache.org/dyn/closer.lua/guacamole/1.6.0/binary/guacamole-1.6.0.war?action=download https://download.java.net/java/GA/jdk25.0.2/b1e0dfa218384cb9959bdcb897162d4e/10/GPL/openjdk-25.0.2_linux-x64_bin.tar.gz https://pub.freerdp.com/releases/freerdp-2.9.0.tar.gz https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.115/bin/apache-tomcat-9.0.115.tar.gz https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-9.6.0.tar.gz
```

## Step 3 Setup the Database

First login into ---> mysql
```
CREATE DATABASE guacamole_db;
CREATE USER 'guacamole_user'@'127.0.0.1' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON guacamole_db.* TO 'guacamole_user'@'127.0.0.1';
FLUSH PRIVILEGES;
EXIT;
```
## Step 4 Install FreeRDP-2.9.0
```
tar -xvf freerdp-2.9.0.tar.gz
mkdir freerdp-2.9.0/build
cd ~/freerdp-2.9.0/build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr -DWITH_CLIENT=OFF -DWITH_URBDRC=OFF ..
make -j$(nproc)
make install
ldconfig
```
## Step 5 Install Apache Guacamole Server
```
mv guacamole-server-1.6.0.tar.gz\?action\=download guacamole-server-1.6.0.tar.gz
tar -xvf guacamole-server-1.6.0.tar.gz
cd guacamole-server-1.6.0
./configure
make
make install
ldconfig
```

## Step 6 Run guacd as a Service (Systemd)
```
chmod +x /usr/local/sbin/guacd <--- make this executable 
```
# Create systemd unit
```
nano /etc/systemd/system/guacd.service
```
```
[Unit]
Description=Guacamole Proxy Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/sbin/guacd -b 0.0.0.0 -l 4822 -f
Restart=on-failure
User=root
Group=root

[Install]
WantedBy=multi-user.target
```
# Reload the Systemd Service
```
systemctl daemon-reload
systemctl enable guacd
systemctl start guacd
```
## Step 7 Create the Guacamole Directories (extension) file

# Create this directories
```
mkdir /etc/guacamole
mkdir /etc/guacamole/extensions
mkdir /etc/guacamole/lib
```
# Create the Properties file
```
nano /etc/guacamole/guacamole.properties
```
```
guacd-hostname: localhost
guacd-port: 4822
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: 123456
mysql-driver: mysql
```
# Move the Connector Files 
```
mv guacamole-auth-jdbc-1.6.0.tar.gz\?action\=download guacamole-auth-jdbc-1.6.0.tar.gz
tar -xvf guacamole-auth-jdbc-1.6.0.tar.gz
cd guacamole-auth-jdbc-1.6.0/mysql
mv guacamole-auth-jdbc-mysql-1.6.0.jar /etc/guacamole/extensions/
```
# Create the Schema
```
cat schema/*.sql | mysql -u root -p guacamole_db
```

# Move the Connector .jar file
```
tar -xvf mysql-connector-j-9.6.0.tar.gz
cd mysql-connector-j-9.6.0
mv mysql-connector-j-9.6.0.jar /etc/guacamole/lib/
```
## Step 8 Install Java 25
```
tar -xvf openjdk-25.0.2_linux-x64_bin.tar.gz
mv jdk-25.0.2/ /opt/Java
export JAVA_HOME=/opt/Java
echo $JAVA_HOME
```

## Step 9 Install Tomcat9
```
tar -xvf apache-tomcat-9.0.115.tar.gz
mv apache-tomcat-9.0.115 /opt/tomcat9
```
```
nano /opt/tomcat9/conf/tomcat-users.xml
```
# Add this lines
```
<role rolename="manager-gui"/>
<user username="admin" password="admin" roles="manager-gui"/>
```
# Remove this line from tomcat9
```
nano /opt/tomcat9/webapps/manager/META-INF/context.xml
```
Remove this line ---> <Valve className="org.apache.catalina.valves.RemoteCIDRValve"

# Turn Tomcat9 into a Service
```
nano /etc/systemd/system/tomcat9.service
```
```
[Unit]
Description=Apache Tomcat 9 Web Application Container
After=network.target

[Service]
Type=forking

# Set Java path
Environment=JAVA_HOME=/opt/Java
Environment=GUACAMOLE_HOME=/etc/guacamole
Environment=CATALINA_PID=/opt/tomcat9/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat9
Environment=CATALINA_BASE=/opt/tomcat9
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat9/bin/startup.sh start
ExecStop=/opt/tomcat9/bin/shutdown.sh stop

User=root
Group=root
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```
# Reload the Systemd Service
```
systemctl daemon-reload
systemctl enable tomcat9
systemctl start tomcat9
```

## Step 10 guacamole.war
```
mv guacamole-1.6.0.war\?action\=download guacamole.war
mv guacamole.war /opt/tomcat9/webapps
```

## Step 11 Verify the Services
```
systemctl restart guacd
systemctl restart tomcat9
systemctl status guacd
systemctl status tomcat9
```

## Step 12 Login in to tomcat9 ---> http://SERVER_IP:8080



