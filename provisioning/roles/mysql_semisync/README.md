Configures an existing mysql installation for semisynchronous replication.

Required variables:

* ```mysql_root_password``` - Used to set up the mysql replication user
* ```mysql_replication_user.name``` - The mysql replication username
* ```mysql_replication_user.password``` - The mysql replication user password
* ```mysql_replication_user.priv``` - The required mysql privileges for the 
replication user. These Should generally be: 
"\*.\*:SELECT,REPLICATION CLIENT,REPLICATION SLAVE,PROCESS,SUPER,RELOAD"
