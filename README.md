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

### 5. Verify the Setup

After running the setup script, you can verify the replication status by checking the MySQL slave status:

```bash
docker exec mysql_slave sh -c "export MYSQL_PWD=$SLAVE_MYSQL_ROOT_PASSWORD; mysql -u root -e 'SHOW SLAVE STATUS \G'"
```
