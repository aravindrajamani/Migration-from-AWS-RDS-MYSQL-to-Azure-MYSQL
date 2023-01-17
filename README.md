# Migration-from-AWS-RDS-MYSQL-to-Azure-MYSQL
Generally we use Data Migration service tools to migrate data from one database to another database but in azure DMS not supporting AWS RDS MYSQL so AWS RDS MYSQL isn't showing in Azure DMS drop down box so we want to manually migrate those data's so let see how to migrate the data from AWS RDS MYSQL to Azure MYSQL.


# About the Company

Our client is a HRMS platform that empowers organizations to manage and retain great talent, Their solutioning approach has enabled lots of organizations to boost their primary metrics by drastically optimizing the employee on-boarding time, increased the turnover ratios and has also helped improvise the help-desk process. Our client provides end to end support platform for streamlining the HR process for any business right from employee on-boarding, payroll, performance review and monitoring employee benefits. They have been  a pioneer in the HR tech space, empowering numerous organizations across the globe to provide the right employee experience.

# The Challenge

The client was already running web applications on AWS RDS instance and the volume of data was huge, around 1 TB. Based on performance bench-marking, the client was looking to migrate the database to Azure on a serverless architecture. Hence it was decided to migrate the database to Azure PaaS with minimal downtime. We had to ensure the end user is least impacted by this activity.

# Exploring possibilities

Generally, one can use AZURE database migration services to migrate data from aws to azure but here that can’t be done because azure migration service does not support online migration from aws rds mysql. So a normal migration cannot be performed.

Hence, any built in tool did not support this kind of migration and an online migration had to be performed to seamlessly migrate the huge volume of data with minimal downtime. An offline migration could not be afforded as the required downtime was close to 18+ hours.

# The Solution 

I had a detailed discussion with the customer and carried out the PoC for the same. The output of the PoC was as follows:
*	The data was exported from the source to the destination server using the export import process.
*	Data-in Replication process was followed which allows the user to synchronize data from any external MySQL server (in our case, the server is AWS RDS) into the Azure Database for MySQL service. The external server can be any of the following 0 a virtual machine, an on-premise server or a database service hosted by any other cloud provider. 
*	The primary usage of this method of replication is when data needs to be kept synchronized between an on-premise server and Azure Database. Or when one needs to keep multiple cloud databases in sync. 
*	The scenario for this client is a complex cloud solution where data has to synchronize between Azure Database for MySQL and database service hosted on AWS RDS. 


Once the POC was successful, I had carried out the process on production environment and the required output was achieved.


# Steps to perform the migration 

Following is a detailed explanation of how to go about with the replication process. 

# STEP 1: (Create Read replica)

One cannot do any activity in the source server (Production server) since there is constant data flow into the source server. Hence a read replica was created in AWS and replication was started from AWS RDS MySQL to AWS read replica with same configuration as the AWS RDS MySQL. 

![image](https://user-images.githubusercontent.com/86965319/212908382-548989d9-2e1d-4c39-a564-9acb821ea915.png)

# STEP 2: (Check Read replica status)

After creating the Read replica in AWS and replication is started, one must Execute Show Slave Status in AWS Read replica and check second_behind_master column in result list.  This needs to be 0 and one can wait a few minutes for the data sync to happen.

Show slave status;

![image](https://user-images.githubusercontent.com/86965319/212908617-aeb4fd8b-8103-4a44-afe0-daf5542155b2.png)

# STEP 3: (Stop Read replica)

AWS Read replica needs to be stopped. This can be done by using call mysql.rds_stop_replication. After stopping replication, slave status can be checked using ‘Execute show slave status’ command.

Following this step, one needs to note down the Relay_Master_log_file and Exect_master_log_position column value. This gives us information regarding the exact position where the replication from the source database to the read replica has been stopped.

CALL mysql.rds_stop_replication;
Show slave status;

Output will be as follows:

![image](https://user-images.githubusercontent.com/86965319/212908883-83653b3f-a3d9-4fcc-bda6-084e2f390d4c.png)
![image](https://user-images.githubusercontent.com/86965319/212908919-92c51112-c3b3-42b3-bba8-18c2dbae3ebd.png)

# STEP 4: (Backup from Read replica)

All the databases needs to be backed up from the AWS Read replica. Hence, a EC2 windows server is created, MySQL server software is installed and all the data is backed up from the AWS Read replica.

![image](https://user-images.githubusercontent.com/86965319/212909086-4375d8ff-678a-44ca-ba90-f3488f6bc546.png)

# STEP 5: (Move backup files to destination)

The backup data is moved from the AWS Read replica to Azure destination database by using any of the file sharing methods.

# STEP 6: (Restore backup file in destination)

The backed up data is restored in the destination database following a similar process as in Step 4. 

![image](https://user-images.githubusercontent.com/86965319/212909428-90be998d-502a-41b6-81ef-210a4430c4c2.png)

# STEP 7: (Delete Read replica)

The read replica in AWS is no more needed and it can be deleted. 

# STEP 8: (Create user in source)

The necessary user with all privileges (mainly replication grants) need to be created in the AWS MySQL server.

Create user user_name@’%’ identified by ‘username’;
Grant replication slave, replication client on *.* to username;

![image](https://user-images.githubusercontent.com/86965319/212909890-dd25f5bf-a489-4e85-808d-b6968822c3c1.png)

# STEP 9: (Replication from source to destination)

This is the last leg of the migration process where the database backup is available in the destination database and hence, data sync from source to destination can be initiated. This ensures data is constantly replicated from AWS RDS MySQL to Azure PAAS MySQL.

CALL mysql.az_replication_change_master('<master_host>', '<master_user>', '<master_password>', <master_port>, '<master_log_file>', <master_log_pos>, '<master_ssl_ca>');

![image](https://user-images.githubusercontent.com/86965319/212910096-a3e2c9ff-37ed-40d3-9ac3-7e7093a3d693.png)

# STEP 10: (Check the status of  Replication)

After replication, to check whether seconds_behind_master column  is 0, show slave status command can be executed. Little bit of latency is expected here and hence the replicated database might need little time to get in sync with the source database. 

Show slave status;

![image](https://user-images.githubusercontent.com/86965319/212910242-98137b93-f87d-4954-b90e-c510222ab28e.png)

# STEP 11: (Testing the destination database)

The table and row counts need to be monitored in the destination database to ensure replication is functional. An added point to be noted here is to use the count function and not the information schema. Reason being, since this database uses InnoDB Database engine, we would get only approximate row count and not be able to get exact row count using the information schema.

For Example:

SELECT SUM(TABLE_ROWS)
   FROM INFORMATION_SCHEMA.TABLES
   WHERE TABLE_SCHEMA = 'yourDatabaseName';

Instead of using above query, use below

(Select ‘Table_Name’,count(*) from ‘Table_Name’) union all
(Select ‘Table_Name’,count(*) from ‘Table_Name’) union all
(Select ‘Table_Name’,count(*) from ‘Table_Name’) union all
(Select ‘Table_Name’,count(*) from ‘Table_Name’) union all
                                               .
                                               .
                                               .
(Select ‘Table_Name’,count(*) from ‘Table_Name’);

# Step 12: (Cutover and Delete RDS)

Once there is no replication lag, based on client confirmation, the IP can be cutover from AWS RDS MySQL to Azure PaaS MySQL. This is the only time during which downtime is required.

Following continuous monitoring, the source database in AWS RDS can be decommissioned. 

# The Benefits

*	The migration process reduced the downtime time required for migration from 20+ hours to 20 minutes.
*	The client was able to switch to the new environment on the same day. As there was live replication happening from source to destination, the method was fail proof.
*	The accelerated migration between different cloud providers optimized performance. It not only optimised the cost and time for the client but also reduced its go-to-market time significantly hence proving to be better ROI. 








