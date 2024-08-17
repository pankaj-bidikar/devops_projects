# Multi Teir Web Application Project Setup

This repository contains the setup guide for the 'Multi Teir Web Application' project, which involves configuring various services using Vagrant, VirtualBox, and other DevOps tools. The guide provides step-by-step instructions for manual provisioning of VMs and configuring essential services like MySQL, Memcached, RabbitMQ, Tomcat, and Nginx.

## Prerequisites

Before starting, ensure you have the following tools installed on your machine:

- **Oracle VM VirtualBox**
- **Vagrant**
- **Vagrant Plugins**
  - Hostmanager Plugin: Install with the following command:
    ```bash
    vagrant plugin install vagrant-hostmanager
    ```
- **Git Bash** or an equivalent editor

## VM Setup

1. Clone the source code repository.
2. Navigate into the repository directory:
    ```bash
    cd /path-to-your-repo/"Multi Teir Web Application"
    ```
3. Switch to the main branch:
    ```bash
    git checkout main
    ```
4. Navigate to the Vagrant directory for manual provisioning:
    ```bash
    cd vagrant/
    ```

5. Bring up the VMs:
    ```bash
    vagrant up
    ```
   **Note:** This process may take some time depending on your system's performance and network speed. If the setup process halts, rerun the command.

## Services Provisioning

The following services are provisioned across multiple VMs:

- **Nginx:** Web Service
- **Tomcat:** Application Server
- **RabbitMQ:** Broker/Queuing Agent
- **Memcached:** Database Caching
- **ElasticSearch:** Indexing/Search Service
- **MySQL:** SQL Database

### Provisioning Order

To ensure proper setup, follow the provisioning order below:

1. **MySQL (Database Service)**
2. **Memcached (Database Caching Service)**
3. **RabbitMQ (Broker/Queue Service)**
4. **Tomcat (Application Service)**
5. **Nginx (Web Service)**

### 1. MySQL Setup

1. SSH into the database VM:
    ```bash
    vagrant ssh db01
    ```
2. Update the system and install necessary packages:
    ```bash
    sudo yum update -y
    sudo yum install epel-release -y
    sudo yum install git mariadb-server -y
    ```
3. Start and enable MariaDB:
    ```bash
    sudo systemctl start mariadb
    sudo systemctl enable mariadb
    ```
4. Secure MariaDB installation:
    ```bash
    sudo mysql_secure_installation
    ```
   Follow the prompts to set the root password and configure security settings.
   - Switch to unix_socket authentication [Y/n] **Y**
   - Change the root password? [Y/n] **Y**
   - Remove anonymous users? [Y/n] **Y**
   - Disallow root login remotely? [Y/n] **N**
   - Remove test database and access to it? [Y/n] **Y**
   - Reload privilege tables now? [Y/n] **Y**

5. Create the database and user:
    ```bash
    mysql -u root -p
    CREATE DATABASE accounts;
    GRANT ALL PRIVILEGES ON accounts.* TO 'admin'@'%' IDENTIFIED BY 'admin123';
    FLUSH PRIVILEGES;
    EXIT;
    ```
6. Initialize the database with the project schema:
    ```bash
    git clone -b main https://github.com/pankaj-bidikar/devops_projects.git
    cd .../"Manual Projects"/"Multi Teir Web Application"
    mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
	mysql -u root -padmin123 accounts
    ```
7. Start the firewall and allow access to MariaDB:
    ```bash
    sudo systemctl start firewalld
	systemctl enable firewalld
	firewall-cmd --get-active-zones
    sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
    sudo firewall-cmd --reload
    sudo systemctl restart mariadb
    ```

### 2. Memcached Setup

1. SSH into the Memcached VM:
    ```bash
    vagrant ssh mc01
    ```
2. Install and start Memcached:
    ```bash
    sudo yum update -y
    sudo dnf install epel-release -y
    sudo dnf install memcached -y
    sudo systemctl start memcached
    sudo systemctl enable memcached
	sudo systemctl status memcached
    ```
3. Configure Memcached to listen on all network interfaces:
    ```bash
    sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
    sudo systemctl restart memcached
    ```
4. Open the firewall for Memcached:
    ```bash
    sudo systemctl start firewalld
	sudo firewall-cmd --add-port=11211/tcp --permanent
	sudo firewall-cmd --runtime-to-permanent
    sudo firewall-cmd --add-port=11111/udp --permanent
	sudo firewall-cmd --runtime-to-permanent
    sudo firewall-cmd --reload
	sudo systemctl status firewalld
    ```

### 3. RabbitMQ Setup

1. SSH into the RabbitMQ VM:
    ```bash
    vagrant ssh rmq01
    ```
2. Install RabbitMQ:
    ```bash
    sudo yum update -y
    sudo yum install epel-release wget -y
	sudo yum Install wget -y
	cd /tmp/
	sudo dnf -y install centos-release-rabbitmq-38
    sudo dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
    sudo systemctl enable --now rabbitmq-server
    ```
3. Create an admin user:
    ```bash
    sudo rabbitmqctl add_user test test
    sudo rabbitmqctl set_user_tags test administrator
    sudo systemctl restart rabbitmq-server
    ```
4. Open the firewall for RabbitMQ:
    ```bash
    sudo firewall-cmd --add-port=5672/tcp --permanent
    sudo firewall-cmd --reload
    ```

### 4. Tomcat Setup

1. SSH into the Tomcat VM:
    ```bash
    vagrant ssh app01
    ```
2. Install Java and dependencies:
    ```bash
    sudo yum update -y
    sudo yum install epel-release -y
    sudo dnf -y install java-11-openjdk java-11-openjdk-devel git maven wget
    ```
3. Download and install Tomcat:
    ```bash
    cd /tmp
    wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.75/bin/apache-tomcat-9.0.75.tar.gz
    tar xzvf apache-tomcat-9.0.75.tar.gz
    sudo useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
    sudo cp -r /tmp/apache-tomcat-9.0.75/* /usr/local/tomcat/
    sudo chown -R tomcat.tomcat /usr/local/tomcat
    ```
4. Set up Tomcat as a service:
    ```bash
    sudo vi /etc/systemd/system/tomcat.service
    ```
   Add the service configuration as provided in the document.

5. Start and enable Tomcat:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start tomcat
    sudo systemctl enable tomcat
    ```

6. Deploy the application:
    ```bash
    git clone -b main https://github.com/pankaj-bidikar/devops_projects.git
    cd "Manual Projects"/"Multi Teir Web Application"
    mvn install
    sudo systemctl stop tomcat
    sudo rm -rf /usr/local/tomcat/webapps/ROOT*
    sudo cp target/'Multi Teir Web Application'-v2.war /usr/local/tomcat/webapps/ROOT.war
    sudo systemctl start tomcat
    ```

### 5. Nginx Setup

1. SSH into the Nginx VM:
    ```bash
    vagrant ssh web01
    ```
2. Install and configure Nginx:
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install nginx -y
    ```
3. Create and configure Nginx site configuration:
    ```bash
    sudo vi /etc/nginx/sites-available/vproapp
    ```
   Add the configuration details provided in the document.

4. Enable the site and restart Nginx:
    ```bash
    sudo ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
    sudo systemctl restart nginx
    ```

## Notes

- Ensure that each service is properly configured and started before moving on to the next.
- Use the provided commands as they are essential for the correct setup and functioning of the 'Multi Teir Web Application' project.

---
## Authors

[@Pankaj bidikar](https://www.github.com/pankaj-bidikar)