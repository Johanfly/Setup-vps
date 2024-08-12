# Setup Ubuntu VPS Laravel + Node + Mysql

### copy ssh from local

``` bash
cat ~/.ssh/id_rsa.pub | ssh root@ipvps "mkdir ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

### Login root

``` bash
ssh root@ipvps
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
adduser johanfly
passwd johanfly
usermod -aG sudo johanfly
exit
```

### copy ssh from local

``` bash
cat ~/.ssh/id_rsa.pub | ssh johanfly@ipvps "mkdir ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

### Secure Ssh

##### update

``` bash
ssh johanfly@ipvps
sudo apt update && sudo apt upgrade -y
sudo chmod 700 ~/.ssh
sudo chmod 600 ~/.ssh/authorized_keys
```

### Lock root
``` bash
sudo passwd -L root
```
##### Install and Configure Google Authenticator

``` bash
sudo apt install -y libpam-google-authenticator
```

###### create & scan a new secret key

``` bash
google-authenticator
```

##### Configure SSH Daemon to Use Google Authenticator

###### Password Authentication with Google Authenticator
``` bash
sudo nano /etc/ssh/sshd_config
```

``` bash
UsePAM yes
ChallengeResponseAuthentication yes
PermitRootLogin yes
```

Save and close the file

``` bash
sudo nano /etc/pam.d/sshd
```

``` bash
@include common-auth
#two-factor authentication via Google Authenticator
auth   required   pam_google_authenticator.so
```

``` bash
sudo systemctl restart ssh
```

###### Public Key Authentication with Google Authenticator

######
``` bash
sudo nano /etc/ssh/sshd_config
```

```
UsePAM yes
ChallengeResponseAuthentication yes
KbdInteractiveAuthentication yes
PasswordAuthentication no
PermitRootLogin no
AuthenticationMethods publickey,keyboard-interactive
```

``` bash
sudo nano /etc/pam.d/sshd
```

``` bash
#@include common-auth
#two-factor authentication via Google Authenticator
auth   required   pam_google_authenticator.so
```

Save and close

``` bash
sudo systemctl restart ssh
```

### Install package

``` bash
sudo apt install zsh git-core curl software-properties-common nginx mysql-server php-cli unzip fail2ban
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.0 php8.0-cli php8.0-common php8.0-mbstring php8.0-xml php8.0-mysql php8.0-curl php8.0-gd php8.0-fpm php8.0-intl php8.0-zip
sudo update-alternatives --config php
```
### Configuring Fail2ban
``` bash
sudo nano /etc/fail2ban/jail.local
```
``` bash
[ssh]
enabled = true
maxretry = 3
findtime = 10
bantime = 1d
```
``` bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Configuring oh-my-zsh & plugin zsh

``` bash
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

``` bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

``` bash
git clone https://github.com/zdharma-continuum/fast-syntax-highlighting.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins/fast-syntax-highlighting
```

### Set plugin zsh

``` bash
nano .zshrc
```

``` bash
plugins=( 
    zsh-autosuggestions
    fast-syntax-highlighting
)
```

### Configuring Ufw

``` bash
sudo nano /etc/default/ufw
```

```
IPV6=yes
```

``` bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw app list
sudo ufw allow OpenSSH
sudo ufw allow "Nginx Full"
sudo ufw allow 2096 #clouflare allow
sudo ufw enable
sudo ufw status
sudo ufw status verbose
```

### Configuring Mysql

``` bash
sudo systemctl start mysql.service
sudo systemctl stop mysql.service
```

``` bash
sudo nano /lib/systemd/system/mysql.service
```

add `--skip-grant-tables --skip-networking`

``` bash
ExecStart=/usr/sbin/mysqld --skip-grant-tables --skip-networking
```

``` bash
sudo systemctl daemon-reload
sudo systemctl start mysql.service
```

``` bash
sudo mysql -u root
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'new_password';
EXIT;
```

``` bash
sudo systemctl stop mysql.service
```

``` bash
sudo nano /lib/systemd/system/mysql.service
```

remove `--skip-grant-tables --skip-networking`

``` bash
sudo systemctl daemon-reload
sudo systemctl start mysql.service
sudo mysql_secure_installation
```

### Install composer

``` bash
cd ~
```

``` bash
curl -sS https://getcomposer.org/installer -o /tmp/composer-setup.php
```

``` bash
HASH=`curl -sS https://composer.github.io/installer.sig`
```

``` bash
echo $HASH
```

``` bash
php -r "if (hash_file('SHA384', '/tmp/composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```

``` bash
sudo php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

### Install Node

``` bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```

``` bash
source .zshrc
```

``` bash
nvm install v16.20.2
nvm use 16.20.2
nvm list-remote
```

### create Mysql db user

``` bash
sudo mysql -u root -p
CREATE DATABASE dbname;
CREATE USER 'dbuser'@'%' IDENTIFIED WITH mysql_native_password BY 'dbpassword';
GRANT ALL ON dbname.* TO 'dbuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

### Set nginx

``` bash
sudo mkdir /var/www/domain.com
sudo chown -R $USER:$USER /var/www/domain.com
```

``` bash
sudo nano /etc/nginx/sites-available/domain.com.conf
```

```
server {
    listen 80;
    listen [::]:80;
    server_name domain.com www.domain.com;

    rewrite ^(.*) https://$host$1/login permanent;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    root /var/www/domain.com/public;
    index index.php;

    ssl_certificate    /etc/ssl/certs/domain.com.pem;
    ssl_certificate_key    /etc/ssl/certs/domain.com.key.pem;


    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.0-fpm.sock;
    }

    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
}
```

``` bash
sudo nginx -t
```

``` bash
sudo ln -s /etc/nginx/sites-available/domain.com.conf /etc/nginx/sites-enabled/domain.com.conf
```

``` bash
sudo systemctl restart nginx
sudo systemctl restart php8.0-fpm
```

### set laravel

``` bash
cd var/www/domain.com
composer update
php artisan key:generate --show
php artisan migrate 
php artisan db:seed
php artisan storage:link
php artisan optimize:clear
npm install
```

`sudo nano server.js`

``` javascript
const https = require('https')
const server = https.createServer({
"key": fs.readFileSync("/etc/ssl/certs/domain.com.key.pem"),
"cert": fs.readFileSync("/etc/ssl/certs/domain.com.pem"),
}, app);
```

``` bash
npm i -g pm2
pm2 start server.js 
pm2 status server.js 
pm2 restart server.js
```

### cron

`crontab -e`

``` bash
* * * * * /usr/bin/php /var/www/domain.com/artisan schedule:run >> /dev/null 2>&1
```

### Fix permission

``` bash
cd /var/www
sudo find domain.com -type f -exec chmod 644 {} \;
sudo find domain.com -type d -exec chmod 755 {} \;
sudo chown -R www-data:www-data /var/www/domain.com
sudo chown -R $USER:$USER /var/www/domain.com/credentials
sudo chmod -R 777 /var/www/domain.com/storage
sudo chmod -R 777 /var/www/domain.com/bootstrap/cache/
```

### Set logrotate
``` bash
sudo nano /etc/conf/mylog.conf
```
``` bash
/var/logs/*.log 
{
    missingok
    notifempty
    daily
    rotate 3
    compress
    delaycompress
    maxage 2
    size 100M
    endscript
}
```
``` bash
sudo logrotate /etc/conf/mylog.conf
```
### Set Cron
``` bash
sudo crontab -e
```
``` bash
0 0 * * * /usr/sbin/logrotate /etc/conf/mylog.conf
```