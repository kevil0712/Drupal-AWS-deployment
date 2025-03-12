# Drupal-AWS-deployment
Comprehensive implementation of a Drupal web application on AWS EC2 using LAMP stack. Features automated installation via Bash scripts, MariaDB configuration, security hardening with iptables, log rotation, scheduled backups, and performance monitoring. Includes detailed step-by-step deployment guide with complete infrastructure documentation.
# Drupal Web Application Deployment on AWS (LAMP Stack)
### Project Overview
This guide details the process of deploying a Drupal web application on an AWS EC2 instance using a LAMP stack (Linux, Apache, MariaDB, PHP). The project includes automated installation and configuration processes via Bash scripting, database setup, user privilege management, network security configurations, and performance monitoring solutions.

### Prerequisites
- AWS account with EC2 access permissions
- Basic knowledge of Linux commands
- SSH client for remote server access
- AWS CLI installed and configured locally (optional but recommended)
- Domain name (optional for production environments)

### Phase 1: AWS Infrastructure Setup

#### 1.1 EC2 Instance Provisioning
1. Log in to AWS Management Console
2. Navigate to EC2 Dashboard
3. Click "Launch Instance"
4. Select Amazon Linux 2 AMI (or Ubuntu Server LTS)
5. Choose instance type:
   - For development: t2.micro (eligible for free tier)
   - For production: t2.medium or higher based on traffic expectations
6. Configure instance details:
   - Network: Default VPC
   - Subnet: Choose an availability zone with lowest latency for your audience
   - Auto-assign Public IP: Enable
7. Add storage:
   - Root volume: 20+ GB gp2 SSD
   - Add additional volume for database backups (optional): 20+ GB gp2 SSD
8. Add tags:
   - Key: Name, Value: Drupal-Web-Server
   - Key: Environment, Value: Production/Development
9. Configure security group:
   - Create new security group: "Drupal-Web-SG"
   - Allow SSH (port 22) from your IP address only
   - Allow HTTP (port 80) from anywhere
   - Allow HTTPS (port 443) from anywhere
10. Review and launch the instance
11. Create or select an existing key pair for SSH access
12. Launch the instance and note the public IP address

#### 1.2 Configure DNS (Optional for Production)
1. In Route 53 (or your domain registrar):
   - Create an A record pointing your domain to the EC2 public IP
   - Create CNAME records for www and other subdomains as needed

### Phase 2: LAMP Stack Installation

#### 2.1 Connect to EC2 Instance
```bash
# Connect via SSH using your key pair
ssh -i /path/to/key-pair.pem ec2-user@your-instance-public-ip
```

#### 2.2 Update System Packages
```bash
# For Amazon Linux 2
sudo yum update -y

# For Ubuntu
# sudo apt update && sudo apt upgrade -y
```

#### 2.3 Install Apache Web Server
```bash
# For Amazon Linux 2
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd

# For Ubuntu
# sudo apt install apache2 -y
# sudo systemctl start apache2
# sudo systemctl enable apache2
```

#### 2.4 Install MariaDB
```bash
# For Amazon Linux 2
sudo amazon-linux-extras install -y mariadb10.5
sudo systemctl start mariadb
sudo systemctl enable mariadb

# For Ubuntu
# sudo apt install mariadb-server mariadb-client -y
# sudo systemctl start mariadb
# sudo systemctl enable mariadb
```

#### 2.5 Secure MariaDB Installation
```bash
sudo mysql_secure_installation
```
Follow the prompts:
- Set root password: Yes
- Remove anonymous users: Yes
- Disallow root login remotely: Yes
- Remove test database: Yes
- Reload privilege tables: Yes

#### 2.6 Install PHP and Required Extensions
```bash
# For Amazon Linux 2
sudo amazon-linux-extras install -y php7.4
sudo yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json php-zip php-posix

# For Ubuntu
# sudo apt install php php-common php-mbstring php-opcache php-cli php-gd php-curl php-mysql php-xml php-json php-intl php-zip -y
```

#### 2.7 Test PHP Installation
```bash
sudo echo "<?php phpinfo(); ?>" > /var/www/html/info.php
# Access http://your-instance-public-ip/info.php in browser to verify
# After verification, remove the file for security
sudo rm /var/www/html/info.php
```

#### 2.8 Configure Apache for PHP
```bash
sudo systemctl restart httpd
# For Ubuntu: sudo systemctl restart apache2
```

### Phase 3: Drupal Installation and Configuration

#### 3.1 Create Database for Drupal
```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE drupal CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'drupaluser'@'localhost' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON drupal.* TO 'drupaluser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

#### 3.2 Download and Extract Drupal
```bash
cd /tmp
wget https://www.drupal.org/download-latest/tar.gz -O drupal.tar.gz
tar -xzf drupal.tar.gz
sudo cp -r drupal-*/* /var/www/html/
sudo cp -r drupal-*/.htaccess /var/www/html/
sudo mkdir -p /var/www/html/sites/default/files
sudo cp /var/www/html/sites/default/default.settings.php /var/www/html/sites/default/settings.php
```

#### 3.3 Set Proper Permissions
```bash
sudo chown -R apache:apache /var/www/html/
sudo chmod -R 755 /var/www/html/
sudo chmod -R 777 /var/www/html/sites/default/files
sudo chmod 666 /var/www/html/sites/default/settings.php
```

#### 3.4 Configure Apache Virtual Host
```bash
sudo nano /etc/httpd/conf.d/drupal.conf
# For Ubuntu: sudo nano /etc/apache2/sites-available/drupal.conf
```

Add the following configuration:
```apache
<VirtualHost *:80>
    ServerName your-domain.com
    ServerAlias www.your-domain.com
    DocumentRoot /var/www/html
    
    <Directory /var/www/html>
        Options FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog /var/log/httpd/drupal_error.log
    CustomLog /var/log/httpd/drupal_access.log combined
</VirtualHost>
```

For Ubuntu, enable the site:
```bash
# sudo a2ensite drupal.conf
# sudo a2enmod rewrite
```

#### 3.5 Restart Apache
```bash
sudo systemctl restart httpd
# For Ubuntu: sudo systemctl restart apache2
```

#### 3.6 Complete Drupal Installation via Browser
1. Navigate to http://your-instance-public-ip or your domain
2. Follow the Drupal installation wizard:
   - Select language
   - Choose installation profile (Standard recommended)
   - Enter database information:
     - Database name: drupal
     - Database username: drupaluser
     - Database password: secure_password
     - Host: localhost
   - Configure site information:
     - Site name
     - Admin username
     - Admin password
     - Admin email
3. Complete the installation

#### 3.7 Secure Drupal Settings
```bash
sudo chmod 644 /var/www/html/sites/default/settings.php
sudo rm /var/www/html/INSTALL.txt /var/www/html/INSTALL.mysql.txt /var/www/html/INSTALL.pgsql.txt /var/www/html/CHANGELOG.txt /var/www/html/COPYRIGHT.txt /var/www/html/MAINTAINERS.txt /var/www/html/LICENSE.txt /var/www/html/INSTALL.sqlite.txt /var/www/html/README.txt
```

### Phase 4: Security Configuration

#### 4.1 Configure Firewall with Iptables
```bash
# Install iptables if not available
sudo yum install -y iptables-services
sudo systemctl start iptables
sudo systemctl enable iptables

# Configure basic firewall rules
sudo iptables -F
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -s your-ip-address/32 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables -A FORWARD -j DROP

# Save iptables configuration
sudo service iptables save
# For Ubuntu: sudo netfilter-persistent save
```

#### 4.2 Harden SSH Configuration
```bash
sudo nano /etc/ssh/sshd_config
```

Make these security changes:
```
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
MaxAuthTries 3
Protocol 2
```

```bash
sudo systemctl restart sshd
```

#### 4.3 Install and Configure Fail2ban
```bash
# For Amazon Linux 2
sudo amazon-linux-extras install -y epel
sudo yum install -y fail2ban
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# For Ubuntu
# sudo apt install fail2ban -y
# sudo systemctl start fail2ban
# sudo systemctl enable fail2ban
```

Create Fail2ban configuration for SSH:
```bash
sudo nano /etc/fail2ban/jail.local
```

Add:
```
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

Restart Fail2ban:
```bash
sudo systemctl restart fail2ban
```

### Phase 5: Automation and Maintenance Scripts

#### 5.1 Create Database Backup Script
```bash
sudo nano /usr/local/bin/backup-drupal.sh
```

Add the following script:
```bash
#!/bin/bash
# Drupal Database Backup Script

# Configuration
BACKUP_DIR="/var/backups/drupal"
MYSQL_USER="drupaluser"
MYSQL_PASSWORD="secure_password"
DATABASE="drupal"
TIMESTAMP=$(date +"%Y%m%d-%H%M%S")

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Database backup
mysqldump -u $MYSQL_USER -p$MYSQL_PASSWORD $DATABASE | gzip > $BACKUP_DIR/drupal-db-$TIMESTAMP.sql.gz

# Files backup
tar -czf $BACKUP_DIR/drupal-files-$TIMESTAMP.tar.gz /var/www/html/sites/default/files

# Delete backups older than 30 days
find $BACKUP_DIR -name "drupal-db-*.sql.gz" -mtime +30 -delete
find $BACKUP_DIR -name "drupal-files-*.tar.gz" -mtime +30 -delete

# Log the backup
echo "Backup completed at $(date)" >> $BACKUP_DIR/backup.log
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/backup-drupal.sh
```

#### 5.2 Schedule Regular Backups with Cron
```bash
sudo crontab -e
```

Add:
```
# Run database backup daily at 2 AM
0 2 * * * /usr/local/bin/backup-drupal.sh
```

#### 5.3 Create Log Rotation Configuration
```bash
sudo nano /etc/logrotate.d/drupal
```

Add:
```
/var/log/httpd/drupal_*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 640 apache apache
    sharedscripts
    postrotate
        /bin/systemctl reload httpd.service > /dev/null 2>/dev/null || true
    endscript
}
```

### Phase 6: Performance Monitoring

#### 6.1 Install and Configure Monitoring Tools
```bash
# Install required tools
sudo yum install -y htop iotop glances

# For Ubuntu
# sudo apt install htop iotop glances -y
```

#### 6.2 Set Up Server Monitoring with AWS CloudWatch
```bash
# Install CloudWatch agent
sudo yum install -y amazon-cloudwatch-agent

# For Ubuntu
# sudo apt install -y amazon-cloudwatch-agent
```

Configure CloudWatch agent:
```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

Add basic configuration:
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "metrics_collected": {
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "resources": [
          "/"
        ]
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ]
      },
      "swap": {
        "measurement": [
          "swap_used_percent"
        ]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/httpd/drupal_error.log",
            "log_group_name": "drupal-error-logs",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/httpd/drupal_access.log",
            "log_group_name": "drupal-access-logs",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```

Start and enable CloudWatch agent:
```bash
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

#### 6.3 Install and Configure Apache Log Analyzer (GoAccess)
```bash
# For Amazon Linux 2
sudo amazon-linux-extras install -y epel
sudo yum install -y goaccess

# For Ubuntu
# sudo apt install goaccess -y
```

Create analysis script:
```bash
sudo nano /usr/local/bin/analyze-logs.sh
```

Add:
```bash
#!/bin/bash
# Generate HTML report from Apache access logs

LOG_DIR="/var/log/httpd"
REPORT_DIR="/var/www/html/reports"
DATE=$(date +"%Y-%m-%d")

# Create reports directory if it doesn't exist
mkdir -p $REPORT_DIR

# Generate report
goaccess $LOG_DIR/drupal_access.log -o $REPORT_DIR/report-$DATE.html --log-format=COMBINED

# Set permissions
chown apache:apache $REPORT_DIR/report-$DATE.html
chmod 644 $REPORT_DIR/report-$DATE.html

echo "Log analysis completed at $(date)" >> $REPORT_DIR/analysis.log
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/analyze-logs.sh
```

Add to crontab:
```bash
sudo crontab -e
```

Add:
```
# Generate traffic reports daily at 3 AM
0 3 * * * /usr/local/bin/analyze-logs.sh
```

### Phase 7: SSL/TLS Configuration (HTTPS)

#### 7.1 Install Certbot for Let's Encrypt SSL
```bash
# For Amazon Linux 2
sudo amazon-linux-extras install -y epel
sudo yum install -y certbot python2-certbot-apache

# For Ubuntu
# sudo apt install certbot python3-certbot-apache -y
```

#### 7.2 Obtain and Configure SSL Certificate
```bash
sudo certbot --apache -d your-domain.com -d www.your-domain.com
```

#### 7.3 Configure Auto-renewal
```bash
# Test renewal process
sudo certbot renew --dry-run

# Certbot creates a cron job automatically, but verify it
sudo ls -la /etc/cron.d/
```

### Phase 8: Performance Optimization

#### 8.1 Install and Configure PHP OPcache
```bash
sudo nano /etc/php.d/10-opcache.ini
# For Ubuntu: sudo nano /etc/php/7.4/apache2/conf.d/10-opcache.ini
```

Add or modify:
```
opcache.enable=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.revalidate_freq=60
opcache.fast_shutdown=1
```

#### 8.2 Configure Apache Performance Settings
```bash
sudo nano /etc/httpd/conf/httpd.conf
# For Ubuntu: sudo nano /etc/apache2/apache2.conf
```

Add or update:
```apache
# Enable server status for monitoring
<Location /server-status>
    SetHandler server-status
    Require ip 127.0.0.1
</Location>

# Performance settings
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

#### 8.3 Install and Configure Redis for Caching (Optional)
```bash
# For Amazon Linux 2
sudo amazon-linux-extras install -y redis4.0
sudo systemctl start redis
sudo systemctl enable redis

# For Ubuntu
# sudo apt install redis-server -y
# sudo systemctl start redis-server
# sudo systemctl enable redis-server
```

Install PHP Redis extension:
```bash
# For Amazon Linux 2
sudo yum install -y php-redis

# For Ubuntu
# sudo apt install php-redis -y
```

Restart Apache:
```bash
sudo systemctl restart httpd
# For Ubuntu: sudo systemctl restart apache2
```

### Phase 9: Documentation and Maintenance Plan

#### 9.1 Create System Documentation File
```bash
sudo nano /root/drupal-system-documentation.md
```

Document:
- Server specifications
- Installed software versions
- Database credentials (stored securely)
- Backup schedule and locations
- Monitoring configurations
- Scheduled maintenance tasks

#### 9.2 Create Maintenance Checklist Script
```bash
sudo nano /usr/local/bin/maintenance-check.sh
```

Add:
```bash
#!/bin/bash
# System maintenance checklist

echo "=== Drupal System Maintenance Check $(date) ===" > /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check disk space
echo "Disk Space Usage:" >> /var/log/maintenance.log
df -h >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check memory usage
echo "Memory Usage:" >> /var/log/maintenance.log
free -m >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check load average
echo "System Load:" >> /var/log/maintenance.log
uptime >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check Apache status
echo "Apache Status:" >> /var/log/maintenance.log
systemctl status httpd | grep Active >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check MariaDB status
echo "MariaDB Status:" >> /var/log/maintenance.log
systemctl status mariadb | grep Active >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check for failed services
echo "Failed Services:" >> /var/log/maintenance.log
systemctl --failed >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check backup status
echo "Latest Backups:" >> /var/log/maintenance.log
ls -la /var/backups/drupal/ | tail -5 >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

# Check for system updates
echo "Available Updates:" >> /var/log/maintenance.log
yum check-update --security | grep -v "^$" | head -20 >> /var/log/maintenance.log
# For Ubuntu: apt list --upgradable | head -20 >> /var/log/maintenance.log
echo "" >> /var/log/maintenance.log

echo "=== Maintenance Check Complete ===" >> /var/log/maintenance.log
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/maintenance-check.sh
```

Schedule weekly maintenance check:
```bash
sudo crontab -e
```

Add:
```
# Run maintenance check weekly on Sunday at 1 AM
0 1 * * 0 /usr/local/bin/maintenance-check.sh
```

### Phase 10: AWS Cost Optimization (Optional)

#### 10.1 Create an Auto Scaling Group (For Production)
1. Create an AMI of your configured EC2 instance
2. Create a launch template or configuration
3. Create an Auto Scaling group with:
   - Minimum: 1 instance
   - Desired: 1 instance
   - Maximum: 2 instances (or as needed)
4. Configure scaling policies based on CPU utilization

#### 10.2 Set Up AWS EC2 Scheduled Actions
Configure scheduled actions to scale down during low-traffic periods and scale up during high-traffic periods.

#### 10.3 Use AWS Reserved Instances
For production environments, consider purchasing Reserved Instances for cost savings on predictable workloads.

---

## Troubleshooting Common Issues

### Issue: Unable to Connect to Database
1. Verify MariaDB is running: `sudo systemctl status mariadb`
2. Check database credentials in Drupal's settings.php
3. Verify database user permissions: 
   ```sql
   SHOW GRANTS FOR 'drupaluser'@'localhost';
   ```

### Issue: Apache Not Starting
1. Check error logs: `sudo tail -f /var/log/httpd/error_log`
2. Verify configuration: `sudo apachectl configtest`
3. Check for port conflicts: `sudo netstat -tulpn | grep :80`

### Issue: File Permission Errors
1. Verify ownership: `ls -la /var/www/html/`
2. Reset permissions: 
   ```bash
   sudo chown -R apache:apache /var/www/html/
   sudo find /var/www/html -type d -exec chmod 755 {} \;
   sudo find /var/www/html -type f -exec chmod 644 {} \;
   sudo chmod -R 777 /var/www/html/sites/default/files
   ```

### Issue: SSL Certificate Errors
1. Verify certificate status: `sudo certbot certificates`
2. Test renewal: `sudo certbot renew --dry-run`
3. Check Apache SSL configuration: `sudo apachectl -t`

---

## Future Enhancements

1. Implement a CDN like AWS CloudFront for global content delivery
2. Set up a staging environment for testing updates before production deployment
3. Implement CI/CD pipeline for automated deployments
4. Migrate database to Amazon RDS for better scalability and management
5. Configure AWS Elastic Load Balancer for high availability
6. Implement AWS WAF (Web Application Firewall) for enhanced security

---

## References

- [AWS Documentation](https://docs.aws.amazon.com/ec2/)
- [Drupal Documentation](https://www.drupal.org/docs)
- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)
- [MariaDB Documentation](https://mariadb.com/kb/en/documentation/)
- [PHP Documentation](https://www.php.net/docs.php)
