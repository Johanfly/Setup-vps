# Setup Ubuntu VPS Laravel + Node + Mysql

### copy ssh from local

``` bash
cat ~/.ssh/id_rsa.pub | ssh root@ipvps "mkdir ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

### Login root

``` bash
ssh root@ipvps
```
``` bash
chmod 700 ~/.ssh
```
``` bash
chmod 600 ~/.ssh/authorized_keys
```
``` bash
adduser johanfly
```
``` bash
passwd johanfly
```
``` bash
usermod -aG sudo johanfly
```

### copy ssh from local

``` bash
cat ~/.ssh/id_rsa.pub | ssh johanfly@ipvps "mkdir ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

### Secure Ssh

##### update

``` bash
ssh johanfly@ipvps
```
``` bash
sudo apt update && sudo apt upgrade -y
```
``` bash
sudo chmod 700 ~/.ssh
```
``` bash
sudo chmod 600 ~/.ssh/authorized_keys
```

### Lock root
``` bash
sudo passwd -l root
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
Save and close

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
sudo apt install zsh git-core curl software-properties-common nginx mysql-server unzip fail2ban
```
``` bash
sudo add-apt-repository ppa:ondrej/php
```
``` bash
sudo apt update
```
``` bash
sudo apt install php8.0 php8.0-cli php8.0-common php8.0-mbstring php8.0-xml php8.0-mysql php8.0-curl php8.0-gd php8.0-fpm php8.0-intl php8.0-zip
```
``` bash
sudo update-alternatives --config php
```
### Configuring Fail2ban
``` bash
sudo nano /etc/fail2ban/jail.local
```
``` bash
[sshd]
enabled = true
maxretry = 3
findtime = 10
bantime = 1d
```
``` bash
sudo systemctl enable fail2ban
```
``` bash
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
```
``` bash
sudo ufw default allow outgoing
```
``` bash
sudo ufw app list
```
``` bash
sudo ufw allow OpenSSH
```
``` bash
sudo ufw allow "Nginx Full"
```
``` bash
sudo ufw allow 2096 #clouflare allow
```
``` bash
sudo ufw enable
```
``` bash
sudo ufw status verbose
```

### Configuring Mysql

``` bash
sudo systemctl start mysql.service
```
``` bash
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
```
``` bash
sudo systemctl start mysql.service
```

``` bash
sudo mysql -u root
```
``` bash
FLUSH PRIVILEGES;
```
``` bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'new_password';
```
``` bash
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
```
``` bash
sudo systemctl start mysql.service
```
``` bash
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
```
``` bash
nvm use 16.20.2
```
``` bash
nvm list-remote
```

### create Mysql db user

``` bash
sudo mysql -u root -p
```
``` bash
CREATE DATABASE dbname;
```
``` bash
CREATE USER 'dbuser'@'%' IDENTIFIED WITH caching_sha2_password BY 'dbpassword';
```
``` bash
GRANT ALL ON dbname.* TO 'dbuser'@'%';
```
``` bash
FLUSH PRIVILEGES;
```
``` bash
EXIT;
```

### Set nginx

``` bash
sudo mkdir /var/www/domain.com
```
``` bash
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
```
``` bash
sudo systemctl restart php8.0-fpm
```

### set laravel

``` bash
cd var/www/domain.com
```
``` bash
composer update
```
``` bash
php artisan key:generate
```
``` bash
php artisan migrate 
```
``` bash
php artisan db:seed
```
``` bash
php artisan storage:link
```
``` bash
php artisan optimize:clear
```
``` bash
npm install
```

``` bash
sudo nano server.js
```

``` javascript
const https = require('https')
const server = https.createServer({
"key": fs.readFileSync("/etc/ssl/certs/domain.com.key.pem"),
"cert": fs.readFileSync("/etc/ssl/certs/domain.com.pem"),
}, app);
```

``` bash
npm i -g pm2
```
``` bash
pm2 start server.js 
```
``` bash
pm2 status server.js
```
``` bash
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
```
``` bash
sudo chown -R www-data:www-data /var/www/domain.com
```
``` bash
sudo find domain.com -type f -exec chmod 644 {} \;
```
``` bash
sudo find domain.com -type d -exec chmod 755 {} \;
```
``` bash
sudo chown -R $USER:$USER /var/www/domain.com/credentials
```
``` bash
sudo chmod -R 777 /var/www/domain.com/storage
```
``` bash
sudo chmod -R 777 /var/www/domain.com/bootstrap/cache/
```

### Set logrotate
``` bash
sudo nano /etc/logrotate.d/mylog.conf
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
}
```
``` bash
sudo logrotate -v /etc/logrotate.d/mylog.conf
```
### Set Cron
``` bash
sudo crontab -e
```
``` bash
0 0 * * * /usr/sbin/logrotate /etc/logrotate.d/mylog.conf
```