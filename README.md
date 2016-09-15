# mysql-ha-pacemaker-semisync

This is a vagrant deployable mysql semisynchronous HA setup with pacemaker as the cluster manager.

It uses udp unicast and corosync encryption as default, but can be configured however you want. 

Install the external roles

    # Install the dependencies
    cd provisioning
    ansible-galaxy install -r requirements.yml -p ./roles
    cd ..

Start the virtual machines if using:

    vagrant plugin install vagrant-hostmanager
    
    # Bring all the machines up before provisioning so the cluster comes up ok on provisioning
    vagrant up --no-provision
        
The servers must be configured with:
* Node1 and Node2 on the same subnet. Node3 can be anywhere. 
* None of the machines can have NAT between them.
* The correct hostnames on all 3 servers
* /etc/hosts set up on all 3 servers and the configuring server
* ports 5404 and 5405 open between all 3 servers
* port 3306 open between Node1 and Node2
* ntp time sync

## Add the first mysl master node
Install the master mysql server on the first node (The Master)
```
ansible-playbook -u root mysql-nodes.yml -i hosts -l node1
```

Remove the anonymous user on the created DB server node1
```
mysql
DELETE FROM mysql.user where User='';
exit
```

## Install the cluster on node1 and the quorum node.
```
mkdir -p /tmp/cluster_setup/corosync/
dd if=/dev/urandom  bs=1K count=1 of=/tmp/cluster_setup/corosync/authkey
ansible-playbook -u root cluster-nodes.yml -i hosts -l node1,node3
```
By now you should have a running cluster, albeit with only one mysql node. 
Use the instructions below to add as many slaves as desired.
    
## Adding a Mysql Slave Node
- First install your ssh key on the existing vms so we can use Ansible manually
- Create the new unprovisioned machine
  ```
  vagrant up node2 --no-provision
  ```
- Install your ssh key on the new machine so you can run ansible manually, and remove the mapping from localaddress to node name
  ```
  vagrant ssh node2
  vim ~/.ssh/authorized_keys
  sudo vim /etc/hosts 
  ```
- Set up the /etc/hosts file on all the machines so they know the new name
  ```
  vagrant provision --provision-with hostmanager
  ```
### Adding the Mysql Aspect 
- Install MySQL on the new node
  ```
  ansible-playbook -u vagrant -b -i 192.168.70.102, \
  -e mysql_server_id=2 \
  -e 'mysql_root_password=s@cure' \
  -e 'replication_user=replication_user' \
  -e 'replication_password=v_s@cure' \
  add-mysql-nodes.yml
  ```
- Stop mysql on the new slave
  ```
  sudo service mysqld stop
  ```
- Install Percona xtrabackup on the master
  ```
  sudo yum install http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
  sudo yum install xtrabackup
  ```
- Backup the Master DB (https://www.percona.com/doc/percona-xtrabackup/2.1/howtos/setting_up_replication.html)
  ```
  mkdir msql_backup
  sudo innobackupex --user=root --password=s@cure msql_backup
  sudo innobackupex --user=root --password=s@cure --apply-log msql_backup/<timestamp>/
  sudo tar zcf msql_backup.tar.gz msql_backup/
  sudo chown vagrant:vagrant msql_backup.tar.gz
  ```
- Copy the backup and the config file from the master to the new slave 
  ```
  scp -3 vagrant@192.168.70.101:/home/vagrant/msql_backup.tar.gz vagrant@192.168.70.102:
  scp -3 vagrant@192.168.70.101:/etc/my.cnf vagrant@192.168.70.102:my.cnf
  ```
- Install the backup at the new slave
  ```
  sudo mv /home/vagrant/my.cnf /etc/my.cnf
  sudo chown mysql:mysql /etc/my.cnf
  # Replace server-id in the newly installed config
  sudo vim /etc/my.cnf
  sudo mv /var/lib/mysql/ /var/lib/mysql_bak
  sudo tar zxf msql_backup.tar.gz
  sudo mv msql_backup/<timestamp>/ /var/lib/mysql
  sudo chown -R mysql:mysql /var/lib/mysql
  ```
- Start mysql at the new slave
  ```
  sudo service mysqld start
  ```
- Set up the replication at the new slave
  ```
  # Cat the binlog info file for the log file and position
  sudo cat /var/lib/mysql/xtrabackup_binlog_info
  sudo mysql
  CHANGE MASTER TO
  MASTER_HOST='192.168.70.101',
  MASTER_USER='replication_user',
  MASTER_PASSWORD='v_s@cure',
  MASTER_LOG_FILE='<current log file>',
  MASTER_LOG_POS=<current log pos>;
  START SLAVE;
  SHOW SLAVE STATUS \G
  exit
  ```
### Adding the cluster aspect
- Upgrade the /etc/cluster/cluster.conf to add the new node 
  - Update the config_version attribute by incrementing its value (for example, changing from config_version="2" to config_version="3">).   
- Copy the updated cluster.conf to all existing cluster nodes, test and validate
  ```
  ansible-playbook -u vagrant -b -i 192.168.70.101,192.168.70.103 -e cman_config=templates/etc/cluster/cluster.conf update-cluster-conf.yml
  ```
- Add the extra node to the pacemaker config
- Install the new slave as a cluster node - This will cause a pacemaker upgrade to the new config.
  ```
  ansible-playbook -u vagrant -b -i 192.168.70.102, \
  -e pacemaker_config_template=templates/pacemaker/cib.xml.j2 \
  -e cman_config_template=templates/etc/cluster/cluster.conf \
  -e corosync_key=/tmp/cluster_setup/corosync/authkey \
  -e replication_user=replication_user \
  -e 'replication_password=v_s@cure' \
  add-cluster-nodes.yml
  ```

# Using the system
See which node is the master ad check the status of the system

    sudo crm_mon -V1

Ssh into the master and create a test db and user
    
    sudo mysql 
    CREATE USER 'test'@'%' IDENTIFIED BY 's@cure';
    GRANT ALL PRIVILEGES ON *.* TO 'test'@'%';
    FLUSH PRIVILEGES;
    exit
    
Then from any host, you can connect via mysql to 192.168.70.200 and it will always hit the active master node.
    
    sudo yum install mysql
    mysql -u test -h 192.168.70.200 -p
    CREATE DATABASE testing;
    CONNECT testing;
    CREATE TABLE pet (name VARCHAR(20), id SERIAL);
    insert into pet (name) values ("fred");

Use this to try and cleanup any resource problems automatically

    sudo pcs resource cleanup p_mysql
