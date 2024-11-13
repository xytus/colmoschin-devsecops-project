Vagrant.configure("2") do |config|

  # Define environment variables directly in the Vagrantfile
  DB_USER = "col_moschin_user"
  DB_PASSWORD = "col_moschin@1234!"
  DB_NAME = "col_moschin_db"
  LUKS_PASSPHRASE = "col_moschin@1234!"

  # Define the Web Server VM
  config.vm.define "web-server" do |web|
    web.vm.box = "ubuntu/bionic64" # Using Ubuntu 18.04 LTS
    web.vm.hostname = "web-server"
    web.vm.network "private_network", ip: "192.168.56.10"

    # Provisioning for Web Server
    web.vm.provision "shell", inline: <<-SHELL
      # Update package list
      sudo apt-get update

      # Install Apache and OpenSSL
      sudo apt-get install -y apache2 openssl
      sudo systemctl enable apache2

      # Generate a Self-Signed SSL Certificate
      sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/ssl/private/apache-selfsigned.key \
        -out /etc/ssl/certs/apache-selfsigned.crt \
        -subj "/C=CA/ST=ON/L=Mississauga/O=LambtonCollege/OU=CSFM/CN=col-moschin-webserver"

      # Configure Apache for HTTPS
      sudo a2enmod ssl
      sudo bash -c 'cat > /etc/apache2/sites-available/default-ssl.conf <<EOF
<VirtualHost *:443>
    ServerAdmin webmaster@localhost
    ServerName col-moschin-webserver
    DocumentRoot /var/www/html
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
</VirtualHost>
EOF'
      sudo a2ensite default-ssl
      sudo systemctl restart apache2

      # Configure Apache with a sample page
      echo "<h1>Col Moschin Web Server</h1>" | sudo tee /var/www/html/index.html

      # Install UFW and configure firewall for Web Server
      sudo apt-get install -y ufw
      sudo ufw default deny incoming
      sudo ufw default allow outgoing
      sudo ufw allow 22/tcp   # Allow SSH from private network only
      sudo ufw allow from 192.168.56.0/24 to any port 80   # Allow HTTP from private network only
      sudo ufw allow from 192.168.56.0/24 to any port 443  # Allow HTTPS from private network only
      echo "y" | sudo ufw enable
      sudo ufw reload

      # Install and configure rsyslog
      sudo apt-get install -y rsyslog
      sudo systemctl enable rsyslog
      sudo systemctl start rsyslog

      # Verify rsyslog configuration and create a test log entry
      logger "Web server logging setup complete"

      # Install Lynis for Security Auditing
      sudo apt-get install -y lynis

      # Run an initial audit with Lynis and save the report
      sudo lynis audit system --quiet --no-colors | sudo tee /var/log/lynis_web_server_audit.log

      # Output the most important findings to the console for review
      sudo grep -E 'warning|suggestion' /var/log/lynis-report.dat > /var/log/lynis_warnings_suggestions.log
      sudo cat  /var/log/lynis_warnings_suggestions.log

      # Display log location
      echo "Lynis Audit Completed. Review  /var/log/lynis_warnings_suggestions.log for details."
    SHELL
  end

  # Define the Database Server VM
  config.vm.define "db-server" do |db|
    db.vm.box = "ubuntu/bionic64"
    db.vm.hostname = "db-server"
    db.vm.network "private_network", ip: "192.168.56.20"

    # Provisioning for Database Server
    db.vm.provision "shell", inline: <<-SHELL
      # Set environment variables
      export DB_USER="#{DB_USER}"
      export DB_PASSWORD="#{DB_PASSWORD}"
      export DB_NAME="#{DB_NAME}"
      export LUKS_PASSPHRASE="#{LUKS_PASSPHRASE}"  
    
      # Update package list
      sudo apt-get update

      # Install MySQL and OpenSSL
      sudo apt-get install -y mysql-server openssl cryptsetup
      sudo systemctl enable mysql

      # Generate SSL Certificates for MySQL
      sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/mysql/mysql-key.pem \
        -out /etc/mysql/mysql-cert.pem \
        -subj "/C=CA/ST=ON/L=Mississauga/O=LambtonCollege/OU=CSFM/CN=db-server"
      sudo openssl dhparam -out /etc/mysql/mysql-dhparams.pem 2048

      # Configure MySQL to Use SSL
      sudo sed -i '/\[mysqld\]/a ssl-ca=/etc/mysql/mysql-dhparams.pem' /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo sed -i '/\[mysqld\]/a ssl-cert=/etc/mysql/mysql-cert.pem' /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo sed -i '/\[mysqld\]/a ssl-key=/etc/mysql/mysql-key.pem' /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo systemctl restart mysql

      # Install UFW and configure firewall for Database Server
      sudo apt-get install -y ufw
      sudo ufw default deny incoming
      sudo ufw default allow outgoing
      sudo ufw allow 22/tcp    # Allow SSH from private network only
      sudo ufw allow from 192.168.56.0/24 to any port 3306  # Allow MySQL from private network only
      echo "y" | sudo ufw enable
      sudo ufw reload

      # Install and configure rsyslog
      sudo apt-get install -y rsyslog
      sudo systemctl enable rsyslog
      sudo systemctl start rsyslog
      
      # Verify rsyslog configuration and create a test log entry
      logger "Database server logging setup complete"

      # Adding Environment Variables for secrets management
           
      # Create a sample database and user using environment variables
      sudo mysql -e "CREATE DATABASE ${DB_NAME};"
      sudo mysql -e "CREATE USER '${DB_USER}'@'%' IDENTIFIED BY '${DB_PASSWORD}';"
      sudo mysql -e "GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'%';"
      sudo mysql -e "FLUSH PRIVILEGES;"

      # Create a new partition
      echo -e "o\nn\np\n1\n\n\nw" | sudo fdisk /dev/sdb

      # Set up LUKS encryption
      echo "${LUKS_PASSPHRASE}" | sudo cryptsetup luksFormat /dev/sdb1

      # Open the encrypted partition
      echo "${LUKS_PASSPHRASE}" | sudo cryptsetup open /dev/sdb1 encrypted_data

      # Format the encrypted partition with ext4
      sudo mkfs.ext4 /dev/mapper/encrypted_data

      # Mount the encrypted filesystem
      sudo mkdir -p /mnt/encrypted_data
      sudo mount /dev/mapper/encrypted_data /mnt/encrypted_data

      # Create a test file in the encrypted filesystem
      echo "This is a test file in encrypted storage" | sudo tee /mnt/encrypted_data/testfile.txt

      # Unmount and close the encrypted partition
      sudo umount /mnt/encrypted_data
      sudo cryptsetup close encrypted_data

      # Install Lynis for Security Auditing
      sudo apt-get install -y lynis

      # Run an initial audit with Lynis and save the report
      sudo lynis audit system --quiet --no-colors | sudo tee /var/log/lynis_db_server_audit.log

      # Output the most important findings to the console for review
      sudo grep -E 'warning|suggestion' /var/log/lynis-report.dat >  /var/log/lynis_warnings_suggestions.log

      sudo cat  /var/log/lynis_warnings_suggestions.log

      # Display log location
      echo "Lynis Audit Completed. Review  /var/log/lynis_warnings_suggestions.log for details."
    SHELL
  end
end
