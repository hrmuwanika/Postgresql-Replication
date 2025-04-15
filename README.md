# PostgreSQL — Physical/Streaming Replication setup

## Build two boxes and install Ubuntu 24.04 LTS

  - box1: master  -> Ubuntu 24.04 LTS with PG 17
    - static ip: 192.168.33.33
    - hostname: master
	
  - box2: slave   -> Ubuntu 24.04 LTS with PG 17
    - static ip: 192.168.33.44
    - hostname: slave


### Configuring the PostgreSQL Master Database Server
     
     #--------------------------------------------------
     # Install Postgresql - Master
     #--------------------------------------------------
     sudo apt update && sudo apt upgrade -y
     sudo apt autoremove
     
     sudo apt install -y nano curl ca-certificates ufw
     sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
     sudo curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
     sudo apt update
     
     sudo apt install -y postgresql-17
     sudo systemctl start postgresql && sudo systemctl enable postgresql

     sudo ufw allow 5432/tcp
	 
  * 1.1. On the master server, switch to the postgres system account and configure the IP address(es) on which the master server will listen to for connections from clients.

    ```
    sed  -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/"  /etc/postgresql/17/main/postgresql.conf
    
    ```
    
  The ALTER SYSTEM SET SQL command is a powerful feature to change a server’s configuration parameters, directly with a SQL query. The configurations are saved in the postgresql.auto.conf file located at the root of data folder (e.g /var/lib/postgresql/17/main) and read addition to those stored in postgresql.conf. But configurations in the former take precedence over those in the later and other related files.

  * 1.2. Then create a replication role that will be used for connections from the standby server to the master server, using the createuser program. In the following command, the -P flag prompts for a password for the new role and -e echoes the commands that createuser generates and sends to the database server.

    ```
    sudo -i -u postgres
    $ psql
    postgres=# CREATE ROLE replicator LOGIN REPLICATION ENCRYPTED PASSWORD 'admin@123';
    postgres=# \q
    exit
    ```

  * 1.3. Then enter the following entry at the end of the /etc/postgresql/17/main/pg_hba.conf client authentication configuration file with the database field set to replication as shown in the screenshot.

    ```
	echo -e "host    replication     replicator            192.168.33.33/32            md5" >> /etc/postgresql/17/main/pg_hba.conf
	echo -e "host       all              all               192.168.33.44/32            md5" >> /etc/postgresql/17/main/pg_hba.conf
    ```

  * 1.4. Now restart the Postgres17 service using the following systemctl command to apply the changes.

    ```
    sudo systemctl restart postgresql
    sudo ufw allow 5432/tcp
    ```

### Configuring the PostgreSQL Standby Database Server

     #--------------------------------------------------
     # Install PostgreSQL - Standby
     #--------------------------------------------------
     sudo apt update && sudo apt upgrade -y
     sudo apt autoremove
     
     sudo apt install -y nano curl ca-certificates ufw
     sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
     sudo curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
     sudo apt update
     
     sudo apt install -y postgresql-17
     sudo systemctl start postgresql && sudo systemctl enable postgresql
	
     sudo ufw allow 5432/tcp

  * 2.1. Next, you need to make a base backup of the master server from the standby server; this helps to bootstrap the standby server. You need to stop the postgresql 17 service on the standby server, switch to the postgres user account, backup the data directory (/var/lib/pgsql/17/data/), then delete everything under it as shown, before taking the base backup.
  
  Standby configuration
  
    ```
    sed  -i "s/#hot_standby = on/hot_standby = on/"  /etc/postgresql/17/main/postgresql.conf
    
    ```

    ```
      systemctl stop postgresql
      cp -R /var/lib/postgresql/17/main/ /var/lib/postgresql/17/main_old/
      rm -rf /var/lib/postgresql/17/main
    ```

  * 2.2. Then use the pg_basebackup tool to take the base backup with the right ownership (the database system user i.e Postgres, within the Postgres user account) and with the right permissions.

    ```
    $ sudo –i -u postgres
    $ pg_basebackup -h 192.168.33.33 -D /var/lib/postgresql/17/main/ -U replicator -P -v -R -X stream -C -S replicator
	
    pg_basebackup: initiating base backup, waiting for checkpoint to complete
    pg_basebackup: checkpoint completed
    pg_basebackup: write-ahead log start point: 0/2000028 on timeline 1
    pg_basebackup: starting background WAL receiver
    pg_basebackup: created replication slot "slaveslot1"
    24506/24506 kB (100%), 1/1 tablespace
    pg_basebackup: write-ahead log end point: 0/2000100
    pg_basebackup: waiting for background process to finish streaming ...
    pg_basebackup: syncing data to disk ...
    pg_basebackup: base backup completed
    
    ```
    ```
    sudo chown postgres:postgres /var/lib/postgresql/17/main/
    
    ```

  In the following command, the option:

    -h – specifies the host which is the master server.
    -D – specifies the data directory.
    -U – specifies the connection user.
    -P – enables progress reporting.
    -v – enables verbose mode.
    -R – enables the creation of recovery configuration: Creates a standby.signal file and append connection settings to postgresql.auto.conf under the data directory.
    -X – used to include the required write-ahead log files (WAL files) in the backup. A value of stream means to stream the WAL while the backup is created.
    -C – enables the creation of a replication slot named by the -S option before starting the backup.
    -S – specifies the replication slot name.

  * 2.3. When the backup process is done, the new data directory on the standby server should look like that in the screenshot. A standby.signal is created and the connection settings are appended to postgresql.auto.conf. You can list its contents using the ls command.

    ```
    # ls -l /var/lib/postgresql/17/main
    total 88
    -rw------- 1 postgres postgres    3 May 31 01:46 PG_VERSION
    -rw------- 1 postgres postgres  224 May 31 01:46 backup_label.old
    drwx------ 6 postgres postgres 4096 May 31 01:51 base
    drwx------ 2 postgres postgres 4096 May 31 01:50 global
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_commit_ts
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_dynshmem
    drwx------ 4 postgres postgres 4096 May 31 01:59 pg_logical
    drwx------ 4 postgres postgres 4096 May 31 01:46 pg_multixact
    drwx------ 2 postgres postgres 4096 May 31 01:49 pg_notify
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_replslot
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_serial
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_snapshots
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_stat
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_stat_tmp
    drwx------ 2 postgres postgres 4096 May 31 01:54 pg_subtrans
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_tblspc
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_twophase
    drwx------ 3 postgres postgres 4096 May 31 01:54 pg_wal
    drwx------ 2 postgres postgres 4096 May 31 01:46 pg_xact
    -rw------- 1 postgres postgres  318 May 31 01:53 postgresql.auto.conf
    -rw------- 1 postgres postgres  130 May 31 01:49 postmaster.opts
    -rw------- 1 postgres postgres  100 May 31 01:49 postmaster.pid
    -rw------- 1 postgres postgres    0 May 31 01:46 standby.signal
    ```

  A replication slave will run in “Hot Standby” mode if the hot_standby parameter is set to on (the default value) in postgresql.conf and there is a standby.signal file present in the data directory.

  * 2.4. Now back on the master server, you should be able to see the replication slot called replicator when you open the pg_replication_slots view as follows.

    ```
    # sudo -i -u postgres
    $ psql -c "SELECT * FROM pg_replication_slots;"
     slot_name  | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn
    ------------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------
     replicator |        | physical  |        |          | f         | f      |            |      |              | 0/2000000   |
    (1 row)
    ```


  * 2.5. To view the connection settings appended in the postgresql.conf file on Slave

    ```
    # nano /var/lib/postgresql/17/main/postgresql.conf
      # Do not edit this file manually!
      # It will be overwritten by the ALTER SYSTEM command.
      listen_addresses = '*'
      standby_mode = 'on'
      primary_conninfo = 'host=192.168.33.33 port=5432 user=replicator password=admin@123'
      primary_slot_name = 'replicator'
      trigger_file = '/var/lib/postgresql/17/main/failover.trigger'
    $ exit
    ```

  * 2.6. Now commence normal database operations on the standby server by starting the PostgreSQL service as follows.

    ```
    # systemctl start postgresql
    ```

### Testing PostgreSQL Streaming Replication

  * 3.1. Once a connection is established successfully between the master and the standby, you will see a WAL receiver process in the standby server with a status of streaming, you can check this using the pg_stat_wal_receiver view.

    ```
    # root@slave:~# sudo -i -u postgres
    # postgres@slave:~$ psql -c "\x" -c "SELECT * FROM pg_stat_wal_receiver;"
      Expanded display is on.
      -[ RECORD 1 ]---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
      pid                   | 5914
      status                | streaming
      receive_start_lsn     | 0/3000000
      receive_start_tli     | 1
      received_lsn          | 0/3000A00
      received_tli          | 1
      last_msg_send_time    | 2020-05-31 02:43:08.441605+00
      last_msg_receipt_time | 2020-05-31 02:43:08.472888+00
      latest_end_lsn        | 0/3000A00
      latest_end_time       | 2020-05-31 01:56:33.422365+00
      slot_name             | pgstandby1
      sender_host           | 192.168.33.33
      sender_port           | 5432
      conninfo              | user=replicator password=******** dbname=replication host=192.168.33.33 port=5432 fallback_application_name=13/main sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
    ```

  and a corresponding WAL sender process in the master/primary server with a state of streaming and a sync_state of async, you can check this pg_stat_replication pg_stat_replication view.

    ```
    $ psql -c "\x" -c "SELECT * FROM pg_stat_replication;"
      Expanded display is on.
      -[ RECORD 1 ]----+------------------------------
      pid              | 6139
      usesysid         | 16384
      usename          | replicator
      application_name | 12/main
      client_addr      | 192.168.33.44
      client_hostname  |
      client_port      | 54930
      backend_start    | 2020-05-31 01:49:30.689483+00
      backend_xmin     |
      state            | streaming
      sent_lsn         | 0/3000A00
      write_lsn        | 0/3000A00
      flush_lsn        | 0/3000A00
      replay_lsn       | 0/3000A00
      write_lag        |
      flush_lag        |
      replay_lag       |
      sync_priority    | 1
      sync_state       | sync
      reply_time       | 2020-05-31 02:44:18.59281+00
    ```

  From the screenshot above, the streaming replication is asynchronous. In the next section, we will demonstrate how to optionally enable synchronous replication.

  * 3.2. Now test if the replication is working fine by creating a test database in the master server and check if it exists in the standby server.

    ```
    # postgres@master:~$ psql
    psql (12.3 (Debian 12.3-1.pgdg18.04+1))
    Type "help" for help.

    # postgres=# create database testdb;
    CREATE DATABASE
    postgres=# \l
                                  List of databases
       Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
    -----------+----------+----------+---------+---------+-----------------------
     postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
     template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres
     template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres
     testdb    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
    (4 rows)
    ```


### Enabling Synchronous Replication on Master

  * Synchronous replication offers the ability to commit a transaction (or write data) to the primary database and the standby/replica simultaneously. It only confirms that a transaction is successful when all changes made by the transaction have been transferred to one or more synchronous standby servers.

  * To enable synchronous replication, the synchronous_commit must also be set to on (which is the default value, thus no need for any change) and you also need to set the synchronous_standby_names parameter to a non-empty value. For this guide, we will set it to all.

    ```
    # postgres@master:~$ psql -c "ALTER SYSTEM SET synchronous_standby_names TO  '*';"
    ALTER SYSTEM
    # postgres@master:/var/lib/postgresql/12/main# cat postgresql.auto.conf
      # Do not edit this file manually!
      # It will be overwritten by the ALTER SYSTEM command.
      listen_addresses = '*'
      synchronous_standby_names = '*'
    ```

  * Then reload the PostgreSQL 17 service to apply the new changes.

    ```
    # systemctl reload postgresql
    ```

  * Now when you query the WAL sender process on the primary server once more, it should show a state of streaming and a sync_state of sync.

    ```
    # postgres@master:~$ psql -c "\x" -c "SELECT * FROM pg_stat_replication;"
    Expanded display is on.
    -[ RECORD 1 ]----+------------------------------
    pid              | 6139
    usesysid         | 16384
    usename          | replicator
    application_name | 13/main
    client_addr      | 192.168.33.44
    client_hostname  |
    client_port      | 54930
    backend_start    | 2020-05-31 01:49:30.689483+00
    backend_xmin     |
    state            | streaming
    sent_lsn         | 0/30019D8
    write_lsn        | 0/30019D8
    flush_lsn        | 0/30019D8
    replay_lsn       | 0/30019D8
    write_lag        |
    flush_lag        |
    replay_lag       |
    sync_priority    | 1
    sync_state       | sync
    reply_time       | 2020-05-31 02:48:43.264218+00
    ```

  We have come to the end of this guide. We have shown how to set up PostgreSQL 13 master-standby database streaming replication in Debian Buster. We also covered how to enable synchronous replication in a PostgreSQL database cluster.

  There are many uses of replication and you can always pick a solution that meets your IT environment and/or application-specific requirements. For more detail, go to [Log-Shipping Standby Servers](https://www.postgresql.org/docs/13/warm-standby.html) in the PostgreSQL 13 documentation.

  * References: https://www.postgresql.org/docs/13/warm-standby.html
