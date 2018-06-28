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
$ mysqldump -u root hive > ${BACKUP_FOLDER}/hive.sql
$ mysqldump -u root hue > ${BACKUP_FOLDER}/hue.sql
```

Capture all Tables schema
Capture all Tables # of records
Remove HA - all components
Backup of CDH configuration files - along with rackawareness
Capture NN, DN, SNN directories path
Push NN in safemode
saveNameSpace
Back up NN image
Back up SNN image
Capture FSCK report 
Review and confirm FSCK results are healthy
Capture LSR report
Capture HDFS report
Capture NN and DN jmx
leave safemode
Shutdown NameNode
Shutdown SecondaryNameNode
Shutdown all DataNodes
Stop all services in CDH



#### Capture all Tables schema
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

```



