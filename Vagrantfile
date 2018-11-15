# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrant API version definition
VAGRANTFILE_API_VERSION = '2'

# VM settings
vm_hostname     = "priscilla.local"
vb_name         = "Priscilla-Dev"
server_memory   = "2048" # MB

@dependencies = <<SCRIPT
# Install dependencies
echo "** [Priscilla] APT Update **\n"
echo 'deb http://packages.dotdeb.org jessie all' >> /etc/apt/sources.list
echo 'deb-src http://packages.dotdeb.org jessie all' >> /etc/apt/sources.list
gpg --keyserver keys.gnupg.net --recv-key 89DF5277
gpg -a --export 89DF5277 | sudo apt-key add -
apt-get update
SCRIPT

@config = <<SCRIPT
echo "** [Priscilla] Add virtual host to hosts file **\n"
echo "127.0.0.1 priscilla.local" >> /etc/hosts

echo "** [Priscilla] Change time zone **\n"
unlink /etc/localtime
ln -s /usr/share/zoneinfo/Europe/London /etc/localtime

echo "** [Priscilla] Change SSH settings **\n"
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service ssh restart
SCRIPT

@server = <<SCRIPT
echo "** [Priscilla] Install Apache2, PHP7 **\n"
apt-get install -y apache2 mc git curl redis-server php7.0 php7.0-bcmath php7.0-bz2 php7.0-dev php7.0-cli php7.0-curl php7.0-intl php7.0-json php7.0-mysql php7.0-mbstring php7.0-opcache php7.0-soap php7.0-sqlite3 php7.0-xml php7.0-gd php7.0-xsl php7.0-zip php7.0-xdebug php7.0-redis libapache2-mod-php7.0
service apache2 restart
SCRIPT

@vhost = <<SCRIPT
echo "** [Priscilla] Create virtual hosts **\n"
echo "
<VirtualHost *:80>
    ServerName priscilla.local
    DocumentRoot /var/www/priscilla/public

    <Directory /var/www/priscilla/public>
        Options +Indexes +FollowSymLinks
        DirectoryIndex index.php index.html
        Order allow,deny
        Allow from all
        AllowOverride All
    </Directory>

    ErrorLog /var/log/apache2/error.log
    CustomLog /var/log/apache2/access.log combined
</VirtualHost>
" > /etc/apache2/sites-available/priscilla.local.conf


echo "** [Priscilla] Enable Apache mods and activate sites **\n"
a2enmod rewrite
a2dissite 000-default
a2ensite priscilla.local.conf
SCRIPT

@xdebug = <<SCRIPT
echo "** [Priscilla] Setting up XDebug **\n"
echo "
xdebug.remote_host=10.0.2.2
xdebug.remote_connect_back=1
xdebug.remote_port=9000
xdebug.remote_enable=1
xdebug.remote_handler=dbgp
xdebug.remote_mode=req
xdebug.idekey=PHPSTORM
" >> /etc/php/7.0/mods-available/xdebug.ini

service apache2 restart
SCRIPT

@database = <<SCRIPT
echo "** [Priscilla] Install and configure MySQL **\n"
sudo debconf-set-selections <<< 'mysql-server mysql-server/root_password password kotor3'
sudo debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password kotor3'
sudo apt-get -y install mysql-server
sed -i "s/.*bind-address.*/#bind-address = 0.0.0.0/" /etc/mysql/my.cnf
sed -i "s/.*skip-external-locking.*/#skip-external-locking/" /etc/mysql/my.cnf
service mysql restart
mysql -uroot -pkotor3 --execute="GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;"
mysql -uroot -pkotor3 --execute="SET PASSWORD FOR 'root'@'%' = PASSWORD('kotor3');"
mysql -uroot -pkotor3 --execute="FLUSH PRIVILEGES"
service mysql restart
SCRIPT

@composer = <<SCRIPT
echo "** [Priscilla] Install Composer dependencies on CRE **\n"
curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
SCRIPT

@project = <<SCRIPT
cd /var/www/priscilla
su vagrant -c "composer install"

echo "** [Priscilla] Creating database **\n"
mysql -u root -pkotor3 -e "CREATE DATABASE priscilla;"
echo "** [Priscilla] Database data migrating **\n"
php /var/www/priscilla/artisan migrate

echo "** [Priscilla] visit http://priscilla.local:8081 in your browser for to view the application **\n"
SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.box = "debian/jessie64"
    config.vm.box_version = "8.7.0"
    config.vm.network "forwarded_port", guest: 80, host: 8081
    config.vm.network "forwarded_port", guest: 443, host: 8443
    config.vm.network "forwarded_port", guest: 3306, host: 33307
    config.vm.hostname = vm_hostname

    if Vagrant::Util::Platform.windows?
        # A private dhcp network is required for NFS to work (on Windows hosts, at least)
        config.vm.network "private_network", type: "dhcp"
        config.vm.synced_folder '.', '/var/www/priscilla', type: "nfs"
        # You MUST have a ~/.ssh/id_rsa SSH key to copy to VM
        if File.exists?(File.join(Dir.home, ".ssh", "id_rsa"))
            # Read local machine's SSH Key (~/.ssh/id_rsa)
            ssh_key = File.read(File.join(Dir.home, ".ssh", "id_rsa"))
            ssh_key_public = File.read(File.join(Dir.home, ".ssh", "id_rsa.pub"))
            # Copy it to VM as the /home/vagrant/.ssh/id_rsa key
            config.vm.provision :shell, :inline => "echo 'Windows-specific: Copying local SSH Key to VM for provisioning...'"
            config.vm.provision :shell, :inline => "mkdir -p /home/vagrant/.ssh"
            config.vm.provision :shell, :inline => "echo '#{ssh_key}' > /home/vagrant/.ssh/id_rsa"
            config.vm.provision :shell, :inline => "echo '#{ssh_key_public}' > /home/vagrant/.ssh/id_rsa.pub"
            config.vm.provision :shell, :inline => "chmod 600 /home/vagrant/.ssh/id_rsa"
            config.vm.provision :shell, :inline => "chmod 600 /home/vagrant/.ssh/id_rsa.pub"
            config.vm.provision :shell, :inline => "chown vagrant -R /home/vagrant/.ssh"
        else
            # Else, throw a Vagrant Error. Cannot successfully startup on Windows without a SSH Key!
            raise Vagrant::Errors::VagrantError, "\n\nERROR: SSH Key not found at ~/.ssh/id_rsa (required on Windows).\nYou can generate this key manually\n\n"
        end
    else
        config.vm.synced_folder '.', '/var/www/priscilla', owner: "www-data", group: "www-data"
    end
    #OS dependencies
    config.vm.provision 'shell', inline: @dependencies
    #OS config
    config.vm.provision 'shell', inline: @config
    #Apache and PHP
    config.vm.provision 'shell', inline: @server
    #Virtual host setup
    config.vm.provision 'shell', inline: @vhost
    #XDebug setup
    config.vm.provision 'shell', inline: @xdebug
    #MySQL setup
    config.vm.provision 'shell', inline: @database
    #Composer installation
    config.vm.provision 'shell', inline: @composer
    #Project configuration
    config.vm.provision 'shell', inline: @project
    config.ssh.forward_agent = true
    config.ssh.username = "vagrant"

    config.vm.provider "virtualbox" do |vb|
      vb.gui = false
      vb.name = vb_name
      vb.customize ["modifyvm", :id, "--memory", server_memory]
    end
end