# pg-failover-repmgr
Project for documenting a PG Master / Slave automated failover install.

This project documents how to setup and configure a Postgres Failover hosts using repmgr for automated failover.

This tutorial draws inspiration from the following pages:
  - prongs.org/blog/postgresql-replication
	- jensd.be/591/linux/setup-a-redundant-postgresql-database-with-repmgr-and-pgpool

This tutorial deploys onto two Centos 7 hosts:
  - pgnode1 - Initially the master node.
	- pgnode2 - Initiall the slave node.

# Requirements

## Install Postgres Database Group Repo to install the PGDG rpms.

    yum -y localinstall http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-1.noarch.rpm
    yum -y install postgresql94-server postgresql94-devel
    yum install -y http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/repmgr94-3.0.2-1.rhel7.x86_64.rpm
   

## Postgres User

The postgres users on both nodes will require password less ssh logins between the nodes for exchanging WAL files and other synchronization tasks.

On both nodes as the postgres user.

    su postgres -
		ssh-keygen
		ssh-copy-id postgres@pgnode2  # And pgnode1 on the second node.
		ls -ld ~/.ssh/
		ls -al ~/.ssh/
		chmod 600 ~/.ssh/
		# If any ssh files are world or group writable fix those.
		# authorized_keys and id_rsa should have 600 perms.

    # Create a repmgr directory for various configs and scripts
		mkdir -p /var/lib/postgres/repmgr/

Test the ssh keys out. It should log you in without prompting for a password.

    ssh postgres@pgnode2

If ssh is prompting for a password. Review the permissions of each postgres user's .ssh directory and files. This is the normal problem with failed ssh key logins.

If you are able to ssh freely between the nodes as postgres then you are ready for the next step.


## The Master Database

    sudo -i
		su postgres -
		/usr/pgsql-9.4/bin/postgresql94-setup initdb
		

### Configure the Host Base Access rules to the database.

Add the following lines to bottom of the /var/lib/pgsql/9.4/data/pg_hba.conf :

    # TYPE    DATABASE    USER      ADDRESS          METHOD
    local     all         all                        trust
    host      all         all       127.0.0.1/32     md5
    host      all         all       ::1/128          md5
    host      repmgr      repmgr    192.168.1.101/32 trust
    host      replication repmgr    192.168.1.101/32 trust
    host      repmgr      repmgr    192.168.1.102/32 trust
    host      replication repmgr    192.168.1.102/32 trust		
    host      all         all       192.168.1.103/32 md5

### postgresql.conf

Set parameters similar to the following at the bottom of the /var/lib/pgsql/9.4/data/postgresql.conf

    listen_addresses = '*'
    max_connections = 200
    shared_buffers = 512MB
    effective_cache_size = 1536MB
    work_mem = 2621kB
    maintenance_work_mem = 128MB
    default_statistics_target = 100
    shared_preload_libraries = 'repmgr_funcs'
    wal_level = hot_standby
    wal_buffers = 16MB
    checkpoint_segments = 32
    checkpoint_completion_target = 0.7
    archive_mode = on
    archive_command = '/var/lib/postgres/repmgr/ship_logs.sh'
    max_wal_senders = 3
    wal_keep_segments = 5000
    wal_sender_timeout = 1s
    hot_standby = on
    log_destination = 'stderr'
    logging_collector = on
    log_directory = 'pg_log' 
    log_filename = 'postgresql-%a.log' 
    log_truncate_on_rotation = on
    log_rotation_age = 1d
    log_rotation_size = 0
    log_min_duration_statement = 0
    log_checkpoints = on
    log_connections = on
    log_disconnections = on
    log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d '
    log_lock_waits = on
    log_statement = 'all'
    log_temp_files = 0
    datestyle = 'iso, mdy'
    lc_messages = 'en_US.UTF-8'
    lc_monetary = 'en_US.UTF-8'
    lc_numeric = 'en_US.UTF-8'
    lc_time = 'en_US.UTF-8'
    default_text_search_config = 'pg_catalog.english'
		shared_preload_libraries = 'repmgr_funcs'

Use http://pgtune.leopard.in.ua/ to tune PG to your connection and hardware needs.

After making these changes.

    sudo systemctl restart postgresql-9.4

### ship_logs.sh

Create the ship_logs.sh script for shipping WAL files off to the slave.

    #!/bin/bash

    # This is called by PG to ship WAL logs to the slave.
    # This is layered on top of replication and is used as another
    # method to secure data successfully transferring from one
    # database server to another.

    ARCHIVE_DIR_ON_SLAVE='/var/lib/pgsql/9.4/walfiles'
    LOG=1
    LOG_FILE=/var/lib/pgsql/9.4/pg_wal_archive.log

    log() {
      echo "$( date --rfc-3339=ns ) $1" >> $LOG_FILE
    }
    log_error() {
      echo "$( date --rfc-3339=ns ) $1" >> $LOG_FILE
      exit 1
    }
    
    wal_path="$1"
    wal_file="$2"
    backup_server="pgnode2"
    
    if [ $LOG -eq 1 ] ; then
      log "Transfer begin of file to $backup_server, filename: $wal_file"
    fi

    rsync "$wal_path" "$backup_server:$ARCHIVE_DIR_ON_SLAVE"
    ret=$?

    if [ $LOG -eq 1 ] ; then
      if [ $ret -eq 0 ] ; then
        log "Transfer to slave complete."
      else
        log_error "Transfer failed $wal_file."
      fi
    fi

### repmgr.conf

Create the following file to configure the repmgr: /var/lib/postgres/repmgr/repmgr.conf

    cluster=pg_cluster
    node=1
    node_name=pgnode1
    conninfo='host=pgnode1 user=repmgr dbname=repmgr'
    pg_bindir=/usr/pgsql-9.4/bin/
    master_response_timeout=5
    reconnect_attempts=2
    reconnect_interval=2
    failover=manual
    promote_command='/usr/pgsql-9.4/bin/repmgr standby promote -f /var/lib/postgres/repmgr/repmgr.conf'
    follow_command='/usr/pgsql-9.4/bin/repmgr standby follow -f /var/lib/postgres/repmgr/repmgr.conf'

Change ownership of the file to postgres

    sudo chown -R postgres.postgres /var/lib/postgres/repmgr/

### Configure firewall

These commands will need to be executed on both pg nodes.

    sudo firewall-cmd --permanent --zone=public --add-service=postgresql
    sudo systemctl reload firewalld

### Create required database users


    sudo -i
    su postgres -
    psql
    postgres# CREATE ROLE repmgr SUPERUSER LOGIN ENCRYPTED PASSWORD 'password';
    postgres# CREATE DATABASE repmgr OWNER repmgr;

### Register the master node

We need to tell repmgr that pgnode1 is the master node currently.

    sudo -i
    su postgres -
    /usr/pgsql-9.4/bin/repmgr -f /var/lib/postgres/repmgr/repmgr.conf master register

You should see a info message similar to this.

    [Notice] master node correctly registered for cluster pg_cluster with id 1 (conninfo host=pgnode1 user=repmgr dbname=repmgr)


## The Slave Node

PG software should be installed. Postgres user should have password less ssh access to the master (pgnode1). Configure the firewall as you did on the master.

### Sync the Slave with the Master

We need to pull the data from the Master so that the slave can follow WAL segments updates.

    sudo -i
    su postgres -
    rm -rf /var/lib/pgsql/9.4/data/*
    /usr/pgsql-9.4/bin/repmgr -D /var/lib/pgsql/9.4/data -d repmgr -p 5432 -U repmgr -R postgres standby clone pgnode1

This step should finish with a message similar to:

    [NOTICE] repmgr standby clone complete

If you need to repeat this step due to errors. Delete the files in the PGDATA dir and try again.

### repmgr.conf

Create the following file to configure the repmgr: /var/lib/postgres/repmgr/repmgr.conf

    cluster=pg_cluster
    node=2
    node_name=pgnode2
    conninfo='host=pgnode2 user=repmgr dbname=repmgr'
    pg_bindir=/usr/pgsql-9.4/bin/
    master_response_timeout=5
    reconnect_attempts=2
    reconnect_interval=2
    failover=manual
    promote_command='/usr/pgsql-9.4/bin/repmgr standby promote -f /var/lib/postgres/repmgr/repmgr.conf'
    follow_command='/usr/pgsql-9.4/bin/repmgr standby follow -f /var/lib/postgres/repmgr/repmgr.conf'

Change ownership of the file to postgres

    sudo chown -R postgres.postgres /var/lib/postgres/repmgr/

### Register the slave with repmgr

    sudo -i
    su postgres -
    /usr/pgsql-9.4/bin/repmgr -f /var/lib/postgres/repmgr/repmgr.conf standby register

The above command should output a message similar to:

    [NOTICE] Standby node correctly registered for cluster pg_cluster with id 2 (conninfo: host=pgnode2 user=repngr dbname=repmgr)

