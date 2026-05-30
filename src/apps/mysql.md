## install

```sh
sudo apt-get install mysql-server
```

## configuration

### password/expose

```sh
sudo mysql -u root -e "SET GLOBAL validate_password.policy=LOW; SET GLOBAL validate_password.length=4; CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED WITH caching_sha2_password BY 'password'; GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION; ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password'; FLUSH PRIVILEGES;"

sudo sed -i 's/^bind-address\s*=\s*127\.0\.0\.1/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf

sudo systemctl restart mysql
```
