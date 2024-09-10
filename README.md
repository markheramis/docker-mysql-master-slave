# MySQL Master-Slave Replication with Docker Compose

This repository contains a setup for a MySQL Master-Slave replication environment using Docker Compose. The setup allows you to easily configure MySQL master-slave replication using environment variables and pre-defined configuration files.

## Prerequisites

- Docker
- Docker Compose
- Bash

## Files

- **docker-compose.yml**: Defines the master and slave MySQL services using the `mysql:5.7` image.
- **.env**: Contains environment variables used to configure both the master and slave MySQL containers.
- **master/conf.d/mysql.conf.cnf**: MySQL configuration file for the master.
- **slave/conf.d/mysql.conf.cnf**: MySQL configuration file for the slave.
- **setup.sh**: Bash script that sets up the MySQL replication.

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/markheramis/docker-mysql-master-slave.git
cd docker-mysql-master-slave
```

### 2. Configure the .env File

Before starting the Docker containers, ensure that the .env file is properly configured with the following variables:

```
# Master MySQL configuration
MASTER_MYSQL_ROOT_PASSWORD=secret
MASTER_MYSQL_PORT=3306
MASTER_MYSQL_USER=homestead
MASTER_MYSQL_PASSWORD=secret
MASTER_MYSQL_DATABASE=homestead
MASTER_MYSQL_LOWER_CASE_TABLE_NAMES=0

# Slave MySQL configuration
SLAVE_MYSQL_ROOT_PASSWORD=secret
SLAVE_MYSQL_PORT=3306
SLAVE_MYSQL_USER=slave_user
SLAVE_MYSQL_PASSWORD=slave_pwd
SLAVE_MYSQL_DATABASE=homestead
SLAVE_MYSQL_LOWER_CASE_TABLE_NAMES=0
```

### 3. Configuration Files

The MySQL configuration files for master and slave (mysql.conf.cnf) can be found in the following locations:

- `master/conf.d/mysql.conf.cnf`: This file is for the master MySQL instance.
- `slave/conf.d/mysql.conf.cnf`: This file is for the slave MySQL instance.

In these files, the `binlog_do_db` directive defines which database is logged and replicated:

```
# Example entry in mysql.conf.cnf
binlog_do_db = homestead
```

The `binlog_do_db` values in these files will be automatically replaced with the values from .env during the setup process. Specifically:

- `MASTER_MYSQL_DATABASE` will replace the value in `master/conf.d/mysql.conf.cnf`.
- `SLAVE_MYSQL_DATABASE` will replace the value in `slave/conf.d/mysql.conf.cnf`.

### 4. Run the Setup Script

To start the setup, use the provided Bash script:

```bash
./setup
```

The script performs the following actions:

1. Stops and removes the existing Docker containers and volumes.
2. Clears the MySQL data directories (master/data and slave/data).
3. Replaces the binlog_do_db entries in the MySQL configuration files based on the .env values.
4. Builds and starts the Docker containers.
5. Waits for the master and slave databases to be ready.
6. Creates a replication user on the master.
7. Configures and starts the replication process on the slave.

The important part in the setup script to note are as follows

- Creation of Replication User in `Master Server`

  The code below creates the `replication_user` which we used in the **slave server** to authenticate to the **master server** as seen [here](https://github.com/markheramis/docker-mysql-master-slave/blob/master/setup#L56).
  ```sql
  CREATE USER "replication_user"@"%" IDENTIFIED BY "replication_pass"; GRANT REPLICATION SLAVE ON *.* TO "replication_user"@"%"; FLUSH PRIVILEGES;
  ```

- Configuring and starting `Slave Server`
  
  The code below will configure the **master server** configuration and start the **slave service** as seen [here](https://github.com/markheramis/docker-mysql-master-slave/blob/master/setup#L71).
  ```sql
  CHANGE MASTER TO MASTER_HOST='mysql_master', MASTER_USER='replication_user', MASTER_PASSWORD='replication_pass', MASTER_LOG_FILE='$CURRENT_LOG', MASTER_LOG_POS=$CURRENT_POS; START SLAVE;
  ```
  **NOTE** that the value of `$CURRENT_LOG` and `$CURRENT_POS` is derived from the result of the `SHOW MASTER STATUS` command executed from the **Master Server** as seen [here](https://github.com/markheramis/docker-mysql-master-slave/blob/master/setup#L63C113-L63C131).

### 5. Verify the Setup

After running the setup script, you can verify that the MySQL replication is functioning correctly. Follow these steps:

#### 5.1 Check the MySQL Slave Replication Status

First, check the status of the replication on the slave database:

```bash
docker exec mysql_slave sh -c "export MYSQL_PWD=$SLAVE_MYSQL_ROOT_PASSWORD; mysql -u root -e 'SHOW SLAVE STATUS \G'"
```

This command should display detailed information about the replication status. Look for the following key fields to verify that replication is working correctly:

- `Slave_IO_Running`: Should be `Yes`.
- `Slave_SQL_Running`: Should be `Yes`.
- `Seconds_Behind_Master`: Should be `0` or a low number, indicating that the slave is in sync with the master.
- `Last_IO_Error` and `Last_SQL_Error`: Should be empty, indicating no errors occurred during replication.

#### 5.2 Test Replication by Performing Operations on the Master

Once the slave status shows no issues, you can test the replication by performing actions on the master database and checking if the changes replicate to the slave.

1. Create a new table on the master:
  ```bash
  docker exec mysql_master sh -c "export MYSQL_PWD=$MASTER_MYSQL_ROOT_PASSWORD; mysql -u root -e 'CREATE DATABASE IF NOT EXISTS test_db; USE test_db; CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255));'"
  ```
2. Insert data into the table:
  ```bash
  docker exec mysql_master sh -c "export MYSQL_PWD=$MASTER_MYSQL_ROOT_PASSWORD; mysql -u root -e 'USE test_db; INSERT INTO users (name) VALUES (\"Alice\"), (\"Bob\"), (\"Charlie\");'"
  ```

#### 5.3 Verify the Data Replication on the Slave

Once the data has been inserted on the master, verify that the changes have been replicated to the slave:

1. Check the existence of the replicated database and table on the slave:
  ```bash
  docker exec mysql_slave sh -c "export MYSQL_PWD=$SLAVE_MYSQL_ROOT_PASSWORD; mysql -u root -e 'SHOW DATABASES;'"
  ```
  You should see test_db in the list of databases.

2. Verify the replicated data by selecting the rows from the users table
  ```bash
  docker exec mysql_slave sh -c "export MYSQL_PWD=$SLAVE_MYSQL_ROOT_PASSWORD; mysql -u root -e 'USE test_db; SELECT * FROM users;'"
  ```
  This should return the following data (or similar):

  ```bash
  +----+---------+
  | id | name    |
  +----+---------+
  |  1 | Alice   |
  |  2 | Bob     |
  |  3 | Charlie |
  +----+---------+
  ```

