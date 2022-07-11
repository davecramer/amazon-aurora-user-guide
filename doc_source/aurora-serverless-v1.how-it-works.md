# How Aurora Serverless v1 works<a name="aurora-serverless-v1.how-it-works"></a>

Following, you can learn how Aurora Serverless v1 works\. 

**Topics**
+ [Aurora Serverless v1 architecture](#aurora-serverless.architecture)
+ [Autoscaling for Aurora Serverless v1](#aurora-serverless.how-it-works.auto-scaling)
+ [Timeout action for capacity changes](#aurora-serverless.how-it-works.timeout-action)
+ [Pause and resume for Aurora Serverless v1](#aurora-serverless.how-it-works.pause-resume)
+ [Determining the maximum number of database connections for Aurora Serverless v1](#aurora-serverless.max-connections)
+ [Parameter groups for Aurora Serverless v1](#aurora-serverless.parameter-groups)
+ [Logging for Aurora Serverless v1](#aurora-serverless.logging)
+ [Aurora Serverless v1 and maintenance](#aurora-serverless.maintenance)
+ [Aurora Serverless v1 and failover](#aurora-serverless.failover)
+ [Aurora Serverless v1 and snapshots](#aurora-serverless.snapshots)

## Aurora Serverless v1 architecture<a name="aurora-serverless.architecture"></a>

 The following image shows an overview of the Aurora Serverless v1 architecture\. 

![\[Aurora Serverless v1 Architecture\]](http://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/images/aurora-serverless-arch.png)

 Instead of provisioning and managing database servers, you specify Aurora capacity units \(ACUs\)\. Each ACU is a combination of approximately 2 gigabytes \(GB\) of memory, corresponding CPU, and networking\. Database storage automatically scales from 10 gibibytes \(GiB\) to 128 tebibytes \(TiB\), the same as storage in a standard Aurora DB cluster\. 

 You can specify the minimum and maximum ACU\. The *minimum Aurora capacity unit* is the lowest ACU to which the DB cluster can scale down\. The *maximum Aurora capacity unit* is the highest ACU to which the DB cluster can scale up\. Based on your settings, Aurora Serverless v1 automatically creates scaling rules for thresholds for CPU utilization, connections, and available memory\. 

 Aurora Serverless v1 manages the warm pool of resources in an AWS Region to minimize scaling time\. When Aurora Serverless v1 adds new resources to the Aurora DB cluster, it uses the router fleet to switch active client connections to the new resources\. At any specific time, you are charged only for the ACUs that are being actively used in your Aurora DB cluster\. 

## Autoscaling for Aurora Serverless v1<a name="aurora-serverless.how-it-works.auto-scaling"></a>

 The capacity allocated to your Aurora Serverless v1 DB cluster seamlessly scales up and down based on the load generated by your client application\. Here, load is CPU utilization and the number of connections\. When capacity is constrained by either of these, Aurora Serverless v1 scales up\. Aurora Serverless v1 also scales up when it detects performance issues that can be resolved by doing so\. 

 You can view scaling events for your Aurora Serverless v1 cluster in the AWS Management Console\. During autoscaling, Aurora Serverless v1 resets the `EngineUptime` metric\. The value of the reset metric value doesn't mean that seamless scaling had problems or that Aurora Serverless v1 dropped connections\. It's simply the starting point for uptime at the new capacity\. To learn more about metrics, see [Monitoring metrics in an Amazon Aurora cluster](MonitoringAurora.md)\. 

 When your Aurora Serverless v1 DB cluster has no active connections, it can scale down to zero capacity \(0 ACUs\)\. To learn more, see [Pause and resume for Aurora Serverless v1](#aurora-serverless.how-it-works.pause-resume)\. 

 When it does need to perform a scaling operation, Aurora Serverless v1 first tries to identify a *scaling point*, a moment when no queries are being processed\. Aurora Serverless v1 might not be able to find a scaling point for the following reasons: 
+  Long\-running queries 
+  In\-progress transactions 
+  Temporary tables or table locks 

 To increase your Aurora Serverless v1 DB cluster's success rate when finding a scaling point, we recommend that you avoid long\-running queries and long\-running transactions\. To learn more about operations that block scaling and how to avoid them, see [Best practices for working with Aurora Serverless v1](http://aws.amazon.com/blogs/database/best-practices-for-working-with-amazon-aurora-serverless/)\. 

 By default, Aurora Serverless v1 tries to find a scaling point for 5 minutes \(300 seconds\)\. You can specify a different timeout period when you create or modify the cluster\. The timeout period can be between 60 seconds and 10 minutes \(600 seconds\)\. If Aurora Serverless v1 can't find a scaling point within the specified period, the autoscaling operation times out\. 

 By default, if autoscaling doesn't find a scaling point before timing out, Aurora Serverless v1 keeps the cluster at the current capacity\. You can change this default behavior when you create or modify your Aurora Serverless v1 DB cluster by selecting the **Force the capacity change** option\. For more information, see [Timeout action for capacity changes](#aurora-serverless.how-it-works.timeout-action)\. 

## Timeout action for capacity changes<a name="aurora-serverless.how-it-works.timeout-action"></a>

 If autoscaling times out without finding a scaling point, by default Aurora keeps the current capacity\. You can choose to have Aurora force the change by selecting the **Force the capacity change** option\. This option is available in the **Autoscaling timeout and action** section of the **Create database** page when you create the cluster\. 

By default, the **Force the capacity change** option isn't selected\. Keep this option clear to have your Aurora Serverless v1 DB cluster's capacity remain unchanged if the scaling operation times out without finding a scaling point\. 

Selecting this option causes your Aurora Serverless v1 DB cluster to enforce the capacity change, even without a scaling point\. Before selecting this option, be aware of the consequences of this selection:
+  Any in\-process transactions are interrupted, and the following error message appears\. 

   **Aurora MySQL 5\.6** – `ERROR 1105 (HY000): The last transaction was aborted due to an unknown error. Please retry.` 

   **Aurora MySQL 5\.7** – `ERROR 1105 (HY000): The last transaction was aborted due to Seamless Scaling. Please retry.` 

   You can resubmit the transactions as soon as your Aurora Serverless v1 DB cluster is available\. 
+  Connections to temporary tables and locks are dropped\. 

  We recommend that you select the **Force the capacity change** option only if your application can recover from dropped connections or incomplete transactions\. 

 The choices that you make in the AWS Management Console when you create an Aurora Serverless v1 DB cluster are stored in the `ScalingConfigurationInfo` object, in the `SecondsBeforeTimeout` and `TimeoutAction` properties\. The value of the `TimeoutAction` property is set to one of the following values when you create your cluster: 
+  `RollbackCapacityChange` – This value is set when you select the **Roll back the capacity change** option\. This is the default behavior\. 
+  `ForceApplyCapacityChange` – This value is set when you select the **Force the capacity change** option\. 

 You can get the value of this property on an existing Aurora Serverless v1 DB cluster by using the [describe\-db\-clusters](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-db-clusters.html) AWS CLI command, as shown following\. 

For Linux, macOS, or Unix:

```
aws rds describe-db-clusters --region region \
  --db-cluster-identifier your-cluster-name \
  --query '*[].{ScalingConfigurationInfo:ScalingConfigurationInfo}'
```

For Windows:

```
aws rds describe-db-clusters --region region ^
  --db-cluster-identifier your-cluster-name ^
  --query "*[].{ScalingConfigurationInfo:ScalingConfigurationInfo}"
```

 As an example, the following shows the query and response for an Aurora Serverless v1 DB cluster named `west-coast-sles` in the US West \(N\. California\) Region\. 

```
$ aws rds describe-db-clusters --region us-west-1 --db-cluster-identifier west-coast-sles 
--query '*[].{ScalingConfigurationInfo:ScalingConfigurationInfo}'

[
    {
        "ScalingConfigurationInfo": {
            "MinCapacity": 1,
            "MaxCapacity": 64,
            "AutoPause": false,
            "SecondsBeforeTimeout": 300,
            "SecondsUntilAutoPause": 300,
            "TimeoutAction": "RollbackCapacityChange"
        }
    }
]
```

 As the response shows, this Aurora Serverless v1 DB cluster uses the default setting\. 

 For more information, see [Creating an Aurora Serverless v1 DB cluster](aurora-serverless.create.md)\. After creating your Aurora Serverless v1, you can modify the timeout action and other capacity settings at any time\. To learn how, see [Modifying an Aurora Serverless v1 DB cluster](aurora-serverless.modifying.md)\. 

## Pause and resume for Aurora Serverless v1<a name="aurora-serverless.how-it-works.pause-resume"></a>

 You can choose to pause your Aurora Serverless v1 DB cluster after a given amount of time with no activity\. You specify the amount of time with no activity before the DB cluster is paused\. When you select this option, the default inactivity time is five minutes, but you can change this value\. This is an optional setting\. 

 When the DB cluster is paused, no compute or memory activity occurs, and you are charged only for storage\. If database connections are requested when an Aurora Serverless v1 DB cluster is paused, the DB cluster automatically resumes and services the connection requests\. 

 When the DB cluster resumes activity, it has the same capacity as it had when Aurora paused the cluster\. The number of ACUs depends on how much Aurora scaled the cluster up or down before pausing it\. 

**Note**  
 If a DB cluster is paused for more than seven days, the DB cluster might be backed up with a snapshot\. In this case, Aurora restores the DB cluster from the snapshot when there is a request to connect to it\. 

## Determining the maximum number of database connections for Aurora Serverless v1<a name="aurora-serverless.max-connections"></a>

The following examples are for an Aurora Serverless v1 DB cluster that's compatible with MySQL 5\.7\. You can use a MySQL client or the query editor, if you've configured access to it\. For more information, see [Running queries in the query editor](query-editor.md#query-editor.running)\.

**To find the maximum number of database connections**

1. Find the capacity range for your Aurora Serverless v1 DB cluster using the AWS CLI\.

   ```
   aws rds describe-db-clusters \
       --db-cluster-identifier my-serverless-57-cluster \
       --query 'DBClusters[*].ScalingConfigurationInfo|[0]'
   ```

   The result shows that its capacity range is 1–4 ACUs\.

   ```
   {
       "MinCapacity": 1,
       "AutoPause": true,
       "MaxCapacity": 4,
       "TimeoutAction": "RollbackCapacityChange",
       "SecondsUntilAutoPause": 3600
   }
   ```

1. Run the following SQL query to find the maximum number of connections\.

   ```
   select @@max_connections;
   ```

   The result shown is for the minimum capacity of the cluster, 1 ACU\.

   ```
   @@max_connections
   90
   ```

1. Scale the cluster to 8–32 ACUs\.

   For more information on scaling, see [Modifying an Aurora Serverless v1 DB cluster](aurora-serverless.modifying.md)\.

1. Confirm the capacity range\.

   ```
   {
       "MinCapacity": 8,
       "AutoPause": true,
       "MaxCapacity": 32,
       "TimeoutAction": "RollbackCapacityChange",
       "SecondsUntilAutoPause": 3600
   }
   ```

1. Find the maximum number of connections\.

   ```
   select @@max_connections;
   ```

   The result shown is for the minimum capacity of the cluster, 8 ACUs\.

   ```
   @@max_connections
   1000
   ```

1. Scale the cluster to the maximum possible, 256–256 ACUs\.

1. Confirm the capacity range\.

   ```
   {
       "MinCapacity": 256,
       "AutoPause": true,
       "MaxCapacity": 256,
       "TimeoutAction": "RollbackCapacityChange",
       "SecondsUntilAutoPause": 3600
   }
   ```

1. Find the maximum number of connections\.

   ```
   select @@max_connections;
   ```

   The result shown is for 256 ACUs\.

   ```
   @@max_connections
   6000
   ```
**Note**  
The `max_connections` value doesn't scale linearly with the number of ACUs\.

1. Scale the cluster back down to 1–4 ACUs\.

   ```
   {
       "MinCapacity": 1,
       "AutoPause": true,
       "MaxCapacity": 4,
       "TimeoutAction": "RollbackCapacityChange",
       "SecondsUntilAutoPause": 3600
   }
   ```

   This time, the `max_connections` value is for 4 ACUs\.

   ```
   @@max_connections
   270
   ```

1. Let the cluster scale down to 2 ACUs\.

   ```
   @@max_connections
   180
   ```

   If you've configured the cluster to pause after a certain amount of time idle, it scales down to 0 ACUs\. However, `max_connections` doesn't drop below the value for 1 ACU\.

   ```
   @@max_connections
   90
   ```

## Parameter groups for Aurora Serverless v1<a name="aurora-serverless.parameter-groups"></a>

 When you create your Aurora Serverless v1 DB cluster, you choose a specific Aurora DB engine and an associated DB cluster parameter group\. Unlike provisioned Aurora DB clusters, an Aurora Serverless v1 DB cluster has a single read/write DB instance that's configured with a DB cluster parameter group only—it doesn't have a separate DB parameter group\. During autoscaling, Aurora Serverless v1 needs to be able to change parameters for the cluster to work best for the increased or decreased capacity\. Thus, with an Aurora Serverless v1 DB cluster, some of the changes that you might make to parameters for a particular DB engine type might not apply\. 

 For example, an Aurora PostgreSQL–based Aurora Serverless v1 DB cluster can't use `apg_plan_mgmt.capture_plan_baselines` and other parameters that might be used on provisioned Aurora PostgreSQL DB clusters for query plan management\. 

 You can get a list of default values for the default parameter groups for the various Aurora DB engines by using the [describe\-engine\-default\-cluster\-parameters](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-engine-default-cluster-parameters.html) CLI command and querying the AWS Region\. The following are values that you can use for the `--db-parameter-group-family` option\. 


|  |  | 
| --- |--- |
|   Aurora MySQL 5\.6   |   `aurora5.6`   | 
|   Aurora MySQL 5\.7   |   `aurora-mysql5.7`   | 
|   Aurora PostgreSQL 10\.12 \(and later\)   |   `aurora-postgresql10`   | 

We recommend that you configure your AWS CLI with your AWS access key ID and AWS secret access key, and that you set your AWS Region before using AWS CLI commands\. Providing the Region to your CLI configuration saves you from entering the `--region` parameter when running commands\. To learn more about configuring AWS CLI, see [Configuration basics](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config) in the *AWS Command Line Interface User Guide*\. 

 The following example gets a list of parameters from the default DB cluster group for Aurora MySQL 5\.6\. 

For Linux, macOS, or Unix:

```
aws rds describe-engine-default-cluster-parameters \
  --db-parameter-group-family aurora5.6 --query \
  'EngineDefaults.Parameters[*].{ParameterName:ParameterName,SupportedEngineModes:SupportedEngineModes} | [?contains(SupportedEngineModes, `serverless`) == `true`] | [*].{param:ParameterName}' \
  --output text
```

For Windows:

```
aws rds describe-engine-default-cluster-parameters ^
   --db-parameter-group-family aurora5.6 --query ^
   "EngineDefaults.Parameters[*].{ParameterName:ParameterName,SupportedEngineModes:SupportedEngineModes} | [?contains(SupportedEngineModes, 'serverless') == `true`] | [*].{param:ParameterName}" ^
   --output text
```

### Modifying parameter values for Aurora Serverless v1<a name="aurora-serverless.parameter-groups.setting-values"></a>

 As explained in [Working with parameter groups](USER_WorkingWithParamGroups.md), you can't directly change values in a default parameter group, regardless of its type \(DB cluster parameter group, DB parameter group\)\. Instead, you create a custom parameter group based on the default DB cluster parameter group for your Aurora DB engine and change settings as needed on that parameter group\. For example, you might want to change some of the settings for your Aurora Serverless v1 DB cluster to [log queries or to upload DB engine specific logs](#aurora-serverless.logging) to Amazon CloudWatch\. 

**To create a custom DB cluster parameter group**

1.  Sign in to the AWS Management Console and then open the Amazon RDS console at [https://console\.aws\.amazon\.com/rds/](https://console.aws.amazon.com/rds/)\. 

1.  Choose **Parameter groups**\. 

1.  Choose **Create parameter group** to open the Parameter group details pane\. 

1.  Choose the appropriate default DB cluster group for the DB engine you want to use for your Aurora Serverless v1 DB cluster\. Be sure that you choose the following options: 

   1.  For **Parameter group family**, choose the appropriate family for your chosen DB engine\. Be sure that your choice has the prefix `aurora-` in its name\. 

   1.  For **Type**, choose **DB Cluster Parameter Group**\. 

   1.  For **Group name** and **Description**, enter meaningful names for you or others who might need to work with your Aurora Serverless v1 DB cluster and its parameters\. 

   1.  Choose **Create**\. 

 Your custom DB cluster parameter group is added to the list of parameter groups available in your AWS Region\. You can use your custom DB cluster parameter group when you create new Aurora Serverless v1 DB clusters\. You can also modify an existing Aurora Serverless v1 DB cluster to use your custom DB cluster parameter group\. After your Aurora Serverless v1 DB cluster starts using your custom DB cluster parameter group, you can change values for dynamic parameters using either the AWS Management Console or the AWS CLI\. 

You can also use the console to view a side\-by\-side comparison of the values in your custom DB cluster parameter group compared to the default DB cluster parameter group, as shown in the following screenshot\. 

![\[Logs published to CloudWatch Logs for Aurora MySQL and Aurora PostgreSQL Aurora Serverless v1 DB clusters\]](http://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/images/aurora-serverless-custom-db-cluster-param-cf-default.png)

 When you change parameter values on an active DB cluster, Aurora Serverless v1 starts a seamless scale in order to apply the parameter changes\. If your Aurora Serverless v1 DB cluster is in a paused state, it resumes and starts scaling so that it can make the change\. The scaling operation for a parameter group change always [forces a capacity change](#aurora-serverless.how-it-works.timeout-action), so be aware that modifying parameters might result in dropped connections if a scaling point can't be found during the scaling period\. 

## Logging for Aurora Serverless v1<a name="aurora-serverless.logging"></a>

 By default, error logs for Aurora Serverless v1 are enabled and automatically uploaded to Amazon CloudWatch\. You can also have your Aurora Serverless v1 DB cluster upload Aurora database\-engine specific logs to CloudWatch\. To do this, enable configuration parameters in your custom DB cluster parameter group\. Your Aurora Serverless v1 DB cluster then uploads all available logs to Amazon CloudWatch\. At this point, you can use CloudWatch to analyze log data, create alarms, and view metrics\. 

 For Aurora MySQL, you can turn on the following logs to have them automatically uploaded from your Aurora Serverless v1 DB cluster to Amazon CloudWatch\. 


|  Aurora MySQL log |  Description  | 
| --- | --- | 
|   `general_log`   |   Creates the general log\. Set to 1 to turn on\. Default is off \(0\)\.   | 
|   `log_queries_not_using_indexes`   |   Logs any queries to the slow query log that don't use an index\. Default is off \(0\)\. Set to 1 to turn on this log\.   | 
|   `long_query_time`   |   Prevents fast\-running queries from being logged in the slow query log\. Can be set to a float between 0 and 3,1536,000\. Default is 0 \(not active\)\.   | 
|   `server_audit_events`   |   The list of events to capture in the logs\. Supported values are `CONNECT`, `QUERY`, `QUERY_DCL`, `QUERY_DDL`, `QUERY_DML`, and `TABLE`\.   | 
|   `server_audit_logging`   |   Set to 1 to turn on server audit logging\. If you turn this on, you can specify the audit events to send to CloudWatch by listing them in the `server_audit_events` parameter\.   | 
|   `slow_query_log`   |   Creates a slow query log\. Set to 1 to turn on the slow query log\. Default is off \(0\)\.   | 

 For more information, see [Using Advanced Auditing with an Amazon Aurora MySQL DB cluster](AuroraMySQL.Auditing.md)\. 

 For Aurora PostgreSQL, you can enable the following logs on your Aurora Serverless v1 DB cluster and have them automatically uploaded to Amazon CloudWatch along with the regular error logs\. 


|  Aurora PostgreSQL log  |  Description  | 
| --- | --- | 
|   `log_connections`   |  Turned on by default and can't be changed\. It logs details for all new client connections\.   | 
|   `log_disconnections`   |   Turned on by default and can't be changed\. Logs all client disconnections\.   | 
|   `log_lock_waits`   |   Default is 0 \(off\)\. Set to 1 to log lock waits\.   | 
|   `log_min_duration_statement`   |   The minimum duration \(in milliseconds\) for a statement to run before it's logged\.   | 
|   `log_min_messages`   |  Sets the message levels that are logged\. Supported values are debug5, debug4, debug3, debug2, debug1, info, notice, warning, error, log, fatal, panic\. To log performance data to the postgres log, set the value to debug1\.  | 
|   `log_temp_files`   |   Logs the use of temporary files that are above the specified kilobytes \(kB\)\.   | 
|   `log_statement`   |   Controls the specific SQL statements that get logged\. Supported values are `none`, `ddl`, `mod`, and `all`\. Default is `none`\.   | 

 After you turn on logs for Aurora MySQL 5\.6, Aurora MySQL 5\.7, or Aurora PostgreSQL for your Aurora Serverless v1 DB cluster, you can view the logs in CloudWatch\. 

### Viewing Aurora Serverless v1 logs with Amazon CloudWatch<a name="aurora-serverless.logging.monitoring"></a>

 Aurora Serverless v1 automatically uploads \("publishes"\) to Amazon CloudWatch all logs that are enabled in your custom DB cluster parameter group\. You don't need to choose or specify the log types\. Uploading logs starts as soon as you enable the log configuration parameter\. If you later disable the log parameter, further uploads stop\. However, all the logs that have already been published to CloudWatch remain until you delete them\. 

 For more information on using CloudWatch with Aurora MySQL logs, see [Monitoring log events in Amazon CloudWatch](AuroraMySQL.Integrating.CloudWatch.md#AuroraMySQL.Integrating.CloudWatch.Monitor)\. 

 For more information about CloudWatch and Aurora PostgreSQL, see [Publishing Aurora PostgreSQL logs to Amazon CloudWatch Logs](AuroraPostgreSQL.CloudWatch.md)\. 

**To view logs for your Aurora Serverless v1 DB cluster**

1. Open the CloudWatch console at [https://console\.aws\.amazon\.com/cloudwatch/](https://console.aws.amazon.com/cloudwatch/)\.

1.  Choose your AWS Region\. 

1.  Choose **Log groups**\. 

1.  Choose your Aurora Serverless v1 DB cluster log from the list\. For error logs, the naming pattern is as follows\. 

   ```
   /aws/rds/cluster/cluster-name/error
   ```

 For example, in the following screenshot you can find listings for logs published for an Aurora PostgreSQL Aurora Serverless v1 DB cluster named `western-sles`\. You can also find several listings for Aurora MySQL Aurora Serverless v1 DB cluster, `west-coast-sles`\. Choose the log that you're interest in to start exploring its content\. 

![\[Logs published to CloudWatch Logs for Aurora MySQL and Aurora PostgreSQL Aurora Serverless v1 DB clusters\]](http://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/images/aurora-serverless-logs-in-cloudwatch.png)

## Aurora Serverless v1 and maintenance<a name="aurora-serverless.maintenance"></a>

 Maintenance for Aurora Serverless v1 DB cluster, such as applying the latest features, fixes, and security updates, is performed automatically for you\. Unlike provisioned Aurora DB clusters, Aurora Serverless v1 doesn't have user\-settable maintenance windows\. However, it does have a maintenance window that you can view in the AWS Management Console in **Maintenance & backups** for your Aurora Serverless v1 DB cluster\. You can find the date and time that maintenance might be performed and if any maintenance is pending for your Aurora Serverless v1 DB cluster, as shown following\. 

![\[Maintenance window for an example Aurora Serverless v1 DB cluster, not user settable, no pending maintenance\]](http://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/images/aurora-serverless-maintenance-window.png)

 Whenever possible, Aurora Serverless v1 performs maintenance in a nondisruptive manner\. When maintenance is required, your Aurora Serverless v1 DB cluster scales its capacity to handle the necessary operations\. Before scaling, Aurora Serverless v1 looks for a scaling point\. It does so for up to seven days if necessary\. 

 At the end of each day that Aurora Serverless v1 can't find a scaling point, it creates a cluster event\. This event notifies you of the pending maintenance and the need to scale to perform maintenance\. The notification includes the date when Aurora Serverless v1 can force the DB cluster to scale\. 

 Until that time, your Aurora Serverless v1 DB cluster continues looking for a scaling point and behaves according to its `TimeoutAction` setting\. That is, if it can't find a scaling point before timing out, it abandons the capacity change if it's configured to `RollbackCapacityChange`\. Or it forces the change if it's set to `ForceApplyCapacityChange`\. As with any change that's forced without an appropriate scaling point, this might interrupt your workload\. 

 For more information, see [Timeout action for capacity changes](#aurora-serverless.how-it-works.timeout-action)\. 

## Aurora Serverless v1 and failover<a name="aurora-serverless.failover"></a>

 If the DB instance for an Aurora Serverless v1 DB cluster becomes unavailable or the Availability Zone \(AZ\) it's in fails, Aurora recreates the DB instance in a different AZ\. However, the Aurora Serverless v1 cluster isn't a Multi\-AZ cluster\. That's because it consists of a single DB instance in a single AZ\. Thus, this failover mechanism takes longer than for an Aurora cluster with provisioned or Aurora Serverless v2 instances\. The Aurora Serverless v1 failover time is undefined because it depends on demand and capacity availability in other AZs within the given AWS Region\. 

 Because Aurora separates computation capacity and storage, the storage volume for the cluster is spread across multiple AZs\. Your data remains available even if outages affect the DB instance or the associated AZ\. 

## Aurora Serverless v1 and snapshots<a name="aurora-serverless.snapshots"></a>

The cluster volume for an Aurora Serverless v1 cluster is always encrypted\. You can choose the encryption key, but you can't disable encryption\. To copy or share a snapshot of an Aurora Serverless v1 cluster, encrypt the snapshot using your own AWS KMS key\. For more information, see [Copying a DB cluster snapshot](aurora-copy-snapshot.md)\. To learn more about encryption and Amazon Aurora, see [Encrypting an Amazon Aurora DB cluster](Overview.Encryption.md#Overview.Encryption.Enabling)