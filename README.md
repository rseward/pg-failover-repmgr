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

Test the ssh keys out. It should log you in without prompting for a password.

    ssh postgres@pgnode2

If ssh is prompting for a password. Review the permissions of each postgres user's .ssh directory and files. This is the normal problem with failed ssh key logins.

If you are able to ssh freely between the nodes as postgres then you are ready for the next step.


## The Master Database

    sudo -i
		su postgres -
		/usr/pgsql-9.4/bin/postgresql94-setup initdb
		

### Configure the Host Base Access rules to the database.

Add the following lines to bottom of the /var/lib/pgsql/9.4/data/ :

    # TYPE    DATABASE    USER      ADDRESS          METHOD
    local     all         all                        trust
    host      all         all       127.0.0.1/32     md5
		host      all         all       ::1/128          md5
		host      repmgr      repmgr    192.168.1.101/32 trust
		host      replication repmgr    192.168.1.101/32 trust
		host      repmgr      repmgr    192.168.1.102/32 trust
		host      replication repmgr    192.168.1.102/32 trust		
		host      all         all       192.168.1.103/32 md5




