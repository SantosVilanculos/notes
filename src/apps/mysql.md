## install

```sh
sudo apt-get install mysql-server
```

## configuration

### password

```sh
sudo mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password'; FLUSH PRIVILEGES;"
```
