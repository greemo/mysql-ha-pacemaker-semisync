# mysql-ha-pacemaker-semisync

This is a vagrant deployable mysql semisynchronous HA setup with pacemaker as the cluster manager.

It uses udp unicast and corosync encryption as default, but can be configured however you want. 

Read the provisioning/README.md as well.

To run:

    # Install the dependencies
    cd provisioning
    ansible-galaxy install -r requirements.yml -p ./roles
    cd ..
    vagrant plugin install vagrant-hostmanager
    
    # Bring all the machines up before provisioning so the cluster comes up ok on provisioning
    vagrant up --no-provision
    vagrant hostmanager
    
    # Then you have to login to each host and remove the node name form /etc/hosts (I know, dodgy)
    vagrant ssh node1 ... sudo vim /etc/hosts ... exit
    ... etc
    
    # We only provision one node, as it does them all anyway.
    vagrant provision node1
    
Then from any host, you can connect via mysql to 192.168.70.200 and it will always hit the active master node.
    
    sudo yum install mysql
    mysql -u replication_user -h 192.168.70.200 -p
    
