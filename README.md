# Setup Ubuntu VPS Laravel + Node + Mysql

### copy ssh from local
```
cat ~/.ssh/id_rsa.pub | ssh root@ipvps "mkdir ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

### Login root
```
ssh root@ip
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
adduser johanfly
passwd johanfly
usermod -aG sudo johanfly
exit
```
### copy ssh from local
```
cat ~/.ssh/id_rsa.pub | ssh johanfly@ipvps "mkdir ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

### Secure Ssh
```
ssh johanfly@ip
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

`sudo nano /etc/ssh/sshd_config`
```
PasswordAuthentication no
PermitRootLogin no
```
```
sudo systemctl restart ssh
```

### Install Zsh Nginx Php Mysql
```
sudo apt aupdate
sudo apt upgrade
sudo apt install zsh git-core curl install software-properties-common nginx mysql-server php-cli unzip
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.0 php8.0-cli php8.0-common php8.0-mbstring php8.0-xml php8.0-mysql php8.0-curl php8.0-gd php8.0-fpm php8.0-intl
sudo update-alternatives --config php
```

### Install oh-my-zsh & plugin zsh
```
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
```
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```
```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

### Set plugin zsh 
`nano .zshrc`
```
plugins=( 
    zsh-autosuggestions
    zsh-syntax-highlighting
)
```
### Set Ufw 

`sudo nano /etc/default/ufw`
```
IPV6=yes
```
```
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
### Set Mysql

```
sudo systemctl start mysql.service
sudo systemctl stop mysql.service
```

`sudo nano /lib/systemd/system/mysql.service`

add `--skip-grant-tables --skip-networking`
```
ExecStart=/usr/sbin/mysqld --skip-grant-tables --skip-networking
```
```
sudo systemctl daemon-reload
sudo systemctl start mysql.service
```
```
sudo mysql -u root
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'new_password';
EXIT;
```
```
sudo systemctl stop mysql.service
```
`sudo nano /lib/systemd/system/mysql.service`

`remove --skip-grant-tables --skip-networking`
```
sudo systemctl daemon-reload
sudo systemctl start mysql.service
sudo mysql_secure_installation
```

### Install composer

```
cd ~
```
```
curl -sS https://getcomposer.org/installer -o /tmp/composer-setup.php
```
```
HASH=`curl -sS https://composer.github.io/installer.sig`
```
```
echo $HASH
```
```
php -r "if (hash_file('SHA384', '/tmp/composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```
```
sudo php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

### Install Node
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```
```
source .zshrc                              
```
```
nvm install v16.20.2
nvm use 16.20.2
nvm list-remote
```
### create Mysql db user
```
sudo mysql -u root -p
CREATE DATABASE dbname;
CREATE USER 'dbuser'@'%' IDENTIFIED WITH mysql_native_password BY 'dbpassword';
GRANT ALL ON dbname.* TO 'dbuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```
### Set nginx
```
sudo mkdir /var/www/domain.com
sudo chown -R $USER:$USER /var/www/domain.com
```
`sudo nano /etc/nginx/sites-available/domain.com.conf`

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
    ssl_certificate_key    /etc/ssl/certs/domain.com.pem;


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


```
sudo nginx -t
```
```
sudo ln -s /etc/nginx/sites-available/domain.com.conf /etc/nginx/sites-enabled/domain.com.conf
```
```
sudo systemctl restart nginx
sudo systemctl restart php8.0-fpm
```

### set laravel
```
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
```
const https = require('https')
const server = https.createServer({
"key": fs.readFileSync("/etc/ssl/certs/domain.com.key.pem"),
"cert": fs.readFileSync("/etc/ssl/certs/domain.com.pem"),
}, app);
```
```
npm i -g pm2
pm2 start server.js 
pm2 status server.js 
pm2 restart server.js 
```
### cron
```
* * * * * /usr/bin/php /var/www/domain.com/artisan schedule:run >> /dev/null 2>&1
```
### Fix permission

```
sudo find domain.com -type f -exec chmod 644 {} \;
sudo find domain.com -type d -exec chmod 755 {} \;
sudo chown -R www-data:www-data /var/www/domain.com
sudo chown -R $USER:$USER /var/www/domain.com/credentials
sudo chmod -R 777 storage
sudo chmod -R 777 bootstrap/cache/
```