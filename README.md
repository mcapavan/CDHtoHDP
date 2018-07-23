# CDHtoHDP
CDH to HDP Migration

Migrating CDH5.7.4 to HDP2.6.5 on CentOS7 and MySQL5.6. Ambari Version is 2.6.2

#### Firewall checks for UI ports, ANCs
taken care by customer
```bash
$ systemctl status firewalld
```
#### All the UI ports opened
```bash
$ telnet <hostname> <port> 
```
#### Check connectivity from Ambari, Hive Nodes to MySQL Server (remote connection) on all Master Nodes*
```bash
$ mysql -h <hostname> -uambari -phadoop
```
#### Clean Up Files, Hive tables and users

#### Validate storage recommendations
```bash
$ df -h
```

#### Announce Cluster Outage
#### Disable service down alarms
#### STOP ETL platform
#### STOP all services like Tableau, Hue, etc for migration readiness
#### Disable IP Tables on all nodes
```bash
$ systemctl stop firewalld
```
#### Backup Hue and Hive Metastore MySQL Database 
```bash
$ mysqldump -uroot -phadoop hive > ${BACKUP_FOLDER}/hive.sql
$ mysqldump -u root -phadoop hue > ${BACKUP_FOLDER}/hue.sql
```

#### Capture all Tables schema and # of records from show create table statement
```bash
hive -e "show databases" | tee databases.txt
cat databases.txt | while read LINE
do
echo $LINE
eval "hive -e 'show tables in $LINE'" | tee $LINE
done

rm -f databases.txt

ls | while read DB_NAME
do
dbname=$DB_NAME
sed -e "s/^/$dbname./" $DB_NAME >> tables.txt
done

cat tables.txt | while read LINE
do
echo $LINE
eval "hive -e 'show create table $LINE'" >> show_create_table.txt
done

```

#### Remove HA in CDH - all components
Please follow the CDH official documentation for removing HA for all components (if any)

https://www.cloudera.com/documentation/enterprise/5-6-x/topics/cdh_hag_hdfs_ha_disabling.html
HDFS - Actions - Disable High Availability
Hive - Actions - Update Hive Metastore Namenodes
Hue - Config - HDFS Web Interface Role - set to NameNode(<hostname>)


#### Backup of CDH configuration files - along with rackawareness

##### copy hadoop config folder from Namenode

cp -pLR /etc/hadoop /tmp/backup/cdh

##### copy hive config folder from Hive node

cp -pLR /etc/hive /tmp/backup/cdh

Note: Rackawareness is available at /etc/hadoop/conf/topology.map


#### Capture NN, DN, SNN directories path from Cloudera Manager
dfs.namenode.name.dir=/dfs/nn
dfs.datanode.data.dir=/dfs/dn
dfs.namenode.checkpoint.dir=/dfs/snn
dfs.journalnode.edits.dir=/var/dfs/jn

Note: Take screenshots if there are too many values


#### Push NN in safemode and saveNameSpace
```bash
$ su hdfs
$ hdfs dfsadmin -safemode enter
$ hdfs dfsadmin -saveNamespace
```

#### Back up NN image
```bash
$ cp -pLR /dfs/nn /tmp/backup/cdh
```
#### Back up SNN image
```bash
$ cp -pLR /dfs/snn /tmp/backup/cdh
```
#### Capture FSCK report 

```bash
$ su hdfs
$ mkdir -p /tmp/backup/cdh/logs
$ hdfs fsck / -files -blocks -locations > /tmp/backup/cdh/logs/dfs-old-fsck-1.log

```
####Review and confirm FSCK results are healthy
```bash
$ tail -25 /tmp/backup/cdh/logs/dfs-old-fsck-1.log
```


#### Capture LSR report
```bash
$ su hdfs
$ hdfs dfs -ls -R / > /tmp/backup/cdh/logs/hdfs-lsr.log
$ tail /tmp/backup/cdh/logs/hdfs-lsr.log
```
#### Capture HDFS report
```bash
$ hdfs dfsadmin -report > /tmp/backup/cdh/logs/dfs-old-report-1.log
$ cat /tmp/backup/cdh/logs/dfs-old-report-1.log
```

#### Capture NN and DN jmx
```bash
$ mkdir -p /tmp/backup/cdh/logs/UI
# for NN:
curl http://pchalla0.field.hortonworks.com:50070/jmx > /tmp/backup/cdh/logs/UI/NN.jmx

# for DN:
for i in {1..3}; do curl http://pchalla$i.field.hortonworks.com:50075/jmx > /tmp/backup/cdh/logs/UI/DN$i.jmx ; done

```

#### Take Screenshorts from NN UI

#### leave safemode
```bash
$ hdfs dfsadmin -safemode leave
```

#### Stop all services in cluster from Cloudera Manager

#### Uninstall PROD CDH cluster from Cloudera Manager
_**` NOTE -- Don't Remove User Data`**_

https://www.cloudera.com/documentation/enterprise/5-8-x/topics/cm_ig_uninstall_cm.html#cmig_topic_18_1_2


#### Create Ambari, hive_temp Databases in MySQL and provide privileges

```bash
mysql -uroot -phadoop

create database hive_temp;
GRANT ALL PRIVILEGES ON *.* TO hive_temp;
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%' IDENTIFIED BY 'hadoop';
FLUSH PRIVILEGES;

CREATE DATABASE ambari_db;
GRANT ALL PRIVILEGES ON *.* TO ambari_db;
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%' IDENTIFIED BY 'hadoop';
FLUSH PRIVILEGES;

```

#### Install Ambari Server

```bash
$ wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.2/ambari.repo -O /etc/yum.repos.d/ambari.repo
$ yum repolist
$ yum install -y ambari-server
$ yum install -y mysql-connector-java
$ ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar

$ # Ambari Server Database Schema setup
$ mysql -h pchalla1.field.hortonworks.com -u ambari -phadoop
  
  USE ambari_db;
  SOURCE /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql;
  \q
```

#### Setup Ambari Server

```bash
$ ambari-server setup
```

#### start Ambari Server

```bash
$ ambari-server start
```

#### Install HDFS (put DUMMY directories), Zookeeper, Ambari Metrics and SmartSense only

#### Can add YARN, Hive, Oozie, Spark post HDFS migration

#### change the NN, DN, SNN directories to restart all services.
#### Expected to fail NN and SNN but not DataNodes
#### Stop all DataNodes
#### Upgrade Namenode

```bash
su -l hdfs
# Add HDFS groups - Can be done in Ambari - HDFS - Add groups
# Add rack awareness for all nodes - one node sample curl command as below

$ curl -u admin:admin -H "X-Requested-By: ambari" -i -X PUT -d '{"Hosts" : {"rack_info" : "/rack0"}}' http://pchalla0.field.hortonworks.com:8080/api/v1/clusters/uds_ref/hosts/pchalla0.field.hortonworks.com

hdfs namenode -upgrade
```

#### Capture FSCK report 

```bash
$ su hdfs
$ mkdir -p /tmp/backup/hdp/logs
$ hdfs fsck / -files -blocks -locations > /tmp/backup/hdp/logs/dfs-new-fsck-1.log

```
####Review and confirm FSCK results are healthy
```bash
$ tail -25 /tmp/backup/hdp/logs/dfs-new-fsck-1.log
```

#### Capture LSR report
```bash
$ su hdfs
$ hdfs dfs -ls -R / > /tmp/backup/hdp/logs/hdfs-lsr.log
$ tail /tmp/backup/hdp/logs/hdfs-lsr.log
```
#### Capture HDFS report
```bash
$ hdfs dfsadmin -report > /tmp/backup/hdp/logs/dfs-new-report-1.log
$ cat /tmp/backup/hdp/logs/dfs-new-report-1.log
```
#### If the reports doesn't match (expect 1-2% variation due to additional directories in HDP), do the rollback as below

```bash
$ namenode -rollback
 ```
#### finalize Upgrade
```bash
$ hdfs dfsadmin -finalizeUpgrade
```

#### Install YARN, MapReduce, Tez and Hive - Point Hive metastore to hive_temp database (as a dummy install)

#### Login to Hive Node and upgrade hive metastore - follow scripts on mysql client

```bash
cd /usr/hdp/2.6.5.0-292/hive2/scripts/metastore/upgrade/mysql
mysql -h pchalla1.field.hortonworks.com -u hive -phadoop hive
source upgrade-1.1.0-to-1.2.0.mysql.sql;
source upgrade-1.2.0-to-1.2.1000.mysql.sql;
source upgrade-1.2.1000-to-2.0.0.mysql.sql;
source upgrade-2.0.0-to-2.1.0.mysql.sql;
source upgrade-2.1.0-to-2.1.1000.mysql.sql;
source upgrade-2.1.1000-to-2.1.2000.mysql.sql;
```

#### Change the Hive database to CDH Hive database of MySQL

#### Recheck all Tables schema and # of records from show create table statement 

#### Migrate saved Hive queries from Hue to Ambari Hive View

Follow https://community.hortonworks.com/articles/198841/migrate-hive-saved-queries-from-hue-390-of-cdh-572.html


