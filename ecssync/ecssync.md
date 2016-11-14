
# ECS-Sync vLab - Migrating data to ECS#
---
DellEMC Elastic Cloud Storage (ECS) is a software-defined, cloud-scale, object storage platform that combines the cost advantages of commodity infrastructure with comprehensive protocol support for unstructured (Object and File) workloads.

ECS-Sync is an open source software tool, developed by DELLEMC engineering, designed to move large amounts of data across the network while maintaining app association and metadata.

It can be used for many use cases:

- Migrating data from a file system to ECS using the S3 API
- Migrating data from Amazon to ECS, pull blobs out of a database and move them into an S3 bucket
- Moving clips from Centera to ECS
- Zip up an Atmos namespace folder into a local archive
- ...

This lab we will be focused on two of the most common use cases that could be valuable in PoCs: File System to ECS and S3 (ECS in our case) to ECS S3 with the corresponding ECS/ECS-Sync configuration files and common troubleshooting actions. 

- 1 - ECS configuration
- 2 - ECS-Sync basic configuration
- 3 - S3 to ECS
- 4 - NFS to ECS

## Prerequisites

### Software


The latest ECS-Sync release is available in the Gihub repository:
https://github.com/EMCECS/ecs-sync/releases

>**Note:** There is also an OVA file that contains a Linux image and the required tools to run ECS-Sync. This VM includes the software required to run ECS-Sync and other tools used for migrating data:

>- Java 1.7
- MySQL
- CAS SDK

>However, this OVA may not come with the latest ECS-Sync version; if that is the case, just copy the new ECS-Sync directory to the */root/ directory* in the ECS-Sync VM and remember to use the latest jar file in your ECS-Sync copy command.

>The OVA file is about 750MB and can be downloaded from [http://bit.ly/ecs-sync-server](http://bit.ly/ecs-sync-server) 

Get the latest ECS-Sync release from your Spark VM:

```
wget https://github.com/EMCECS/ecs-sync/releases/download/v3.0-beta.2/ecs-sync-3.0-beta.2.zip
unzip ecs-sync-3.0-beta.2.zip
```

If Java is not installed, please do it.

```
yum install java
```

###Network

ECS-Sync uses the network to copy data from the source to the target. So, the ECS-Sync VM should have excellent connectivity to both the source and the target of the copy, preferably 10 Gigabit Ethernet.

### Load Balancers 

In most of cases, no load balancer is required when using ECS-Sync. From ECS-Sync 2.0, it uses the new ECS S3 plugin with the smart-client feature. This plugin provides client-side load-balancing, node discovery, VDC awareness and health checking. More details about this plugin can be found at
[https://community.emc.com/docs/DOC-49430#jive_content_id_The_Smart_Client](https://community.emc.com/docs/DOC-49430#jive_content_id_The_Smart_Client)

In the case of CAS being the source or the target of the copy, no load balancer is required since the CAS SDK already includes internal load balancing itself.

### ECS

As the usual target of ECS-Sync copy jobs, we need to create a bucket (with the desired features) in ECS. Note that the bucket can be created from ECSSync, but it is a best practice to do it from the ECS, having full control of the features enabled in that bucket.

ECS-Sync will need this information:

- Endpoints (ECS nodes)
        - One node is enough if using the ECS S3 plugin with the smart-client
- Bucket
	- FS enabled if using NFS/HDFS
- Credentials
	- User /Secret Key for S3
	- pea file for CAS
	- Subtenant ID for Atmos

If you don't have access to an ECS system, you can create an account on [ECS Test Drive](http://portal.ecstestdrive.com).

For this lab, please use the ECS F&F deployed in your lab:

- ecs-1-1.vlab.local: 192.168.1.11
- ecs-1-2.vlab.local: 192.168.1.12
- ecs-1-3.vlab.local: 192.168.1.13

# Lab

## 1 - ECS Configuration

- Create an ECS object user called *ecssync_user1* and get its secret key. 
- Create an ECS object user called *ecssync_user2* and get its secret key. 
- Create a bucket called *bucketssource*, own by *ecssync_user1* 
    - Upload a few objects to this bucket
- Create a bucket with FS disabled called *buckettargets3*, owned by *ecssync_user2*
- Create a bucket with FS enabled called *bucketteargetnfs*, owned by *ecssync_user2*

## 2 - ECS-Sync Configuration

The ECS-Sync configuration will vary depending on the nature of the data you copy.

There are several ECS-Sync execution paths:

- UI
	- ECS-Sync 2.1 provides a browser-based UI for simple filesystem-to-ECS archiving operations
- Command line
	- All plugins are self-documenting and self-configuring so they can be executed from the command line
- Spring Integration
	- An entire plugin chain can be configured using the supplied Spring bootstrapper or loaded into an existing Spring application
- Custom Development
-	 Plugins can be configured and executed at runtime from any Java application (a *toolbox* for reuse in other applications)

We'll use mainly ECS-Sync CLI and ECS-Sync UI in this lab.

- For the ECS-Sync CLI we will use the latest and greatest version directly on your Spark VM: *ecs-sync 3.0 beta 2*.
- For the ECS-Sync UI we will use *ecs-sync v2.1.2* in a Docker Container, since it is not available in *ecs-sync 3.0*yet.

# 3 - ECS-Sync copy tasks

This section will cover the most popular ECS-Sync copies.

- File System to ECS S3
- S3 to ECS S3

## 3.1 - File System to ECS S3

- Source: File System
- Target: ECS S3 bucket

Directories and files are processed as they are listed. So, first will be the root folder and all its children, then the children of all the 1st-level directories and so on.

Directories are preserved in S3 as empty objects.  All directories and files will preserve their *mtime/atime/crtime* and *POSIX owner/group/mode*. These features can be viewed as user-metadata on the objects starting with *x-emc-(mtime|atime|crtime) and x-emc-posix-(owner-name|group-owner-name|mode)*.  When syncing back to the filesystem, ECS-Sync will attempt to restore these attributes (so, you need to run as *root* and mount as a *root-host*).

Subdirectories can be excluded from the copy using the *--exclude-paths <pattern,pattern,...>* option. This uses proper regular expressions, so you would have to specify something like *\.snapshot* in order to exclude the snapshot subdirectory from the copy.

When copying from NFS, it is also useful to run *iozone* on the mount in order to find the optimal thread count.

### 3.1.1 - Using XML file

XML samples files are located in the *sample* directory, in your ECS-Sync directory.
 
We'll use the *directory-to-ecs-s3.xml* file as a template for the FS to ECS S3 copy.

```
cat sample/directory-to-ecs-s3-test.xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<syncConfig xmlns="http://www.emc.com/ecs/sync/model" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.emc.com/ecs/sync/model model.xsd">
    <syncOptions>
        <threadCount>32</threadCount>
        <verify>true</verify>
        <logLevel>verbose</logLevel>
    </syncOptions>

    <source>
        <filesystemConfig>
            <path>/root/ecs-sync-3.0-beta.2/sample/</path>
        </filesystemConfig>
    </source>

    <target>
        <awsS3Config>
            <host>192.168.1.11</host><!-- defaults to HTTPS // Added Port and Protocol to force HTTP instead-->
            <port>9020</port>
            <protocol>http</protocol>
            <accessKey>ecssync_user2</accessKey>
            <secretKey>6gRNVQ9MQuElCPQrZcCZ0mcjKvwJOTrahiI8H2S1</secretKey>
            <disableVHosts>true</disableVHosts>
            <legacySignatures>true</legacySignatures>
            <bucketName>bucketteargetnfs</bucketName>
            <createBucket>false</createBucket>
            <!-- default behavior is to preserve directories (including empty ones) for later restoration -->
            <preserveDirectories>false</preserveDirectories>
        </awsS3Config>
    </target>
</syncConfig>

```

This ECS-Sync job copies data from the File System */root/ecs-sync-3.0-beta.2/sample/* to the S3 ECS bucket *bucketteargetnfs*.

Source | Target
--- | ---
Filesystem on the ECS-Sync VM | ECS S3 Bucket
/root/ecs-sync-3.0-beta.2/sample/ | bucketteargetnfs

Execute the ECS-Sync copy:
`java -jar ecs-sync-3.0-beta.2.jar --xml-config sample/directory-to-ecs-s3-test.xml`

```
EcsSync v3.0-beta.2
2016-11-10 08:50:50 WARN  [main           ] RestServer: REST server listening at http://localhost:9200/
Transferred 34,050 bytes in 1 seconds (34,050 bytes/s)
Successful files: 12 (12/s) Failed Files: 0
Failed files: []
```

### 3.1.2 - ECS-Sync UI

ECS-Sync also provides a browser–based User Interface (UI) for ease of use from version 2.1, but not available yet for version 3.0. For this lab we will use ecs-sync-ui-2.1.2. Please refer to *Appendix 1 -  Launching a ECS-Sync UI Docker Container in the vLab* and launch an ECS-Sync UI cont	ainer. Create a bucket with FS enabled called *bucketteargetnfsui*, owned by *ecssync_user2*

ECS-Sync UI provides:

- Filesystem (incl. mounted NFS/CIFS) and ECS S3 plugins
- Supports multiple simultaneous syncs with progress and error reporting
- Archives sync completion and error reports
- Options for multiple sync schedules
- Email notifications (start, complete, error)
- Configuration stored in ECS - VM is disposable

Once your ECS-Sync UI instance is launched, configure the ECS-Sync copy parameters. Go to *Config*, indicate the *ECS endpoints*, *ECS User = ecssync_user2* and *Secret Key = ecssync_user2_secret_key*, and an email for alerts. Click *Save & Write Configuration to ECS*.

>**Note:** In the current ECS-Sync UI version a bucket called *ecs-sync* will be created to save the ECS-Sync UI configuration. This bucket cannot be used as a target for the ECS-Sync copy.

![UI1](ui1_new.jpg)

```
>**Note:**When using the ECS-Sync OVA, if you get a *Service  Unavailable* when lauching the ECS-Sync UI, please make sure the services are started:

`sudo service ecs-sync start && sudo service ecs-sync-ui start`

```

Once the configuration is successfully saved, go to *Schedule* and configure a schedule for the copy.

- Indicate */tmp/nfssource* as the *Filesystem Source Directory*
- Indicate *bucketteargetnfsui* as the *ECS S3 Target Bucket*

![UI3](ui3_new.jpg)
 

Clicking in *show advanced options*, you can select the *Log Level* or the *DB type* among other options. 

>**Note:** Please, don't use DB in the vLab to avoid potential issues with this light ECS-Sync-UI image. There is no problem configuring the DB when using the regular ECS-Sync OVA.

The *Schedule* tab shows all the Scheduled ECS-Sync copies in the ECS-Sync VM.

 ![UI5](ui5_new.jpg)

There is also a *Reports* tab that shows all the copy reports.

### 3.1.3 - Isilon Backup

ECS-Sync can be used as a tool for backup. It is easy to copy a NAS share to ECS to provide a low cost archiving data protection on a different media using the ECS-Sync UI.

Isilon data (NFS) can be periodically copied to ECS (S3) by running regular incremental ECS-Sync copies.

 ![Isilon](isilon.jpg)

## 3.2 - S3 to ECS S3

This section will show how to perform a copy from a S3 bucket to an ECS S3 bucket. 

In this case, since we cannot use Amazon as a source, we'll copy from a S3 ECS bucket to another S3 ECS bucket. 

We'll use the *s3-to-ecs-s3.xml* file as a template for the S3 to ECS S3 copy.

```
cat sample/s3-to-ecs-s3-test.xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!--This is a sample configuration to migrate an AWS S3 bucket to an ECS S3 bucket, including all object versions. It uses 32 threads, verifies data using MD5 checksums and tracks status of all objects in a database table.-->
<syncConfig xmlns="http://www.emc.com/ecs/sync/model"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://www.emc.com/ecs/sync/model model.xsd">
    <options>
        <threadCount>32</threadCount>
        <verify>true</verify>
    </options>

    <source>
        <awsS3Config>
            <host>192.168.1.11</host><!-- defaults to HTTPS -->
            <port>9020</port>
            <protocol>http</protocol>
            <accessKey>ecssync_user1</accessKey>
            <secretKey>cJv394eB8/DheamTK/iuutuCsz/f5HWUknMtW//H</secretKey>
            <disableVHosts>true</disableVHosts>
            <legacySignatures>true</legacySignatures>
            <bucketName>bucketssource</bucketName>
            <createBucket>false</createBucket>
        </awsS3Config>
    </source>

    <target><!-- TODO: change this to ecsS3Config when available -->
        <awsS3Config>
            <host>192.168.1.11</host><!-- defaults to HTTPS -->
            <port>9020</port>
            <protocol>http</protocol>
            <accessKey>ecssync_user2</accessKey>
            <secretKey>6gRNVQ9MQuElCPQrZcCZ0mcjKvwJOTrahiI8H2S1</secretKey>
            <disableVHosts>true</disableVHosts>
            <legacySignatures>true</legacySignatures>
            <bucketName>buckettargets3</bucketName>
            <createBucket>false</createBucket>
        </awsS3Config>
    </target>
</syncConfig>

```

Modify the XML file in order to configure a copy from the S3 ECS Bucket *bucketssource*, owned by *ecssync_user1* to *buckettargets3*, owned by *ecssync_user2*

Source | Target 
--- | ---
ECS S3 Bucket | ECS S3 Bucket 
bucketssource | buckettargets3 

Populate the ECS S3 Bucket *bucketssource* with a few objects, so that you can actually copy data.

Execute the ECS-Sync copy:

`java -jar ecs-sync-3.0-beta.2.jar --xml-config sample/s3-to-ecs-s3-test.xml`

```
EcsSync v3.0-beta.2
2016-11-10 12:07:36 WARN  [main           ] RestServer: REST server listening at http://localhost:9200/
Transferred 10,744,882 bytes in 1 seconds (10,744,882 bytes/s)
Successful files: 6 (6/s) Failed Files: 0
Failed files: []
```

## 3.3 - Managing the ECS-Sync copy 

### 3.3.1 - Modifying the ECS-Sync copy using *ecs-sync-ctl-xx.jar*

As part of the ECS-Sync suite, there is a REST service that allows interaction over HTTP to *start*, *pause*, *resume* and *stop* sync jobs as well as change thread counts and provide progress information.

This *ecs-sync-ctl-xx.jar* file (previously called *ecs-sync-cli-xx.jar*) can be downloaded from the GitHub website:

[https://github.com/EMCECS/ecs-sync/releases](https://github.com/EMCECS/ecs-sync/releases).

It should already be available in your ecs-sync home directory.

These are *ecs-sync-ctl-xx.jar* options to manage, modify or monitor an ECS-Sync copy:

```
usage: java -jar ecs-sync-ctl-{version}.jar --delete <job-id> | --list-jobs | --pause <job-id> | --resume <job-id> | --set-threads <job-id> | --status <job-id> | --stop <job-id> | --submit <xml-file> [--endpoint <url>]  [--log-file <filename>] [--log-level <level>] [--log-pattern <log4j-pattern>]  [--query-threads <thread-count>]      [--sync-threads <thread-count>]

```

| Option | Description |
| ------------- | ------------- | 
| --delete <job-id> | Deletes a sync job from the server. The job must be stopped first. | 
| --endpoint <url> | Sets the server endpoint to connect to.  Default is http://localhost:9200 |
| --list-jobs | Lists jobs in the server |
| --log-file <filename> | Filename to write log messages. Setting to STDOUT or STDERR will write log messages to the  appropriate  process stream.  Default is STDERR. |
| --log-level <level> | Sets the log level: DEBUG, INFO, WARN, ERROR, or FATAL.  Default is ERROR. |
| --log-pattern <log4j-pattern> | Sets the Log4J pattern to use when writing log messages.  Defaults to  %d{yyyy-MM-dd HH:mm:ss}%-5p[%t] %c{1}:%L - %m%n |
| --pause <job-id> | Pauses the specified sync job |
| --query-threads <thread-count> | Used in conjunction with --set-threads to set the number of query threads to use for a job. |
| --resume <job-id> | Resumes a specified sync job |
| --set-threads <job-id> | Sets the number of sync threads on the server.  Requires --query-threads and/or --sync-threads arguments |
| --status <job-id> | Queries the server for job status |
| --stop <job-id> | Terminates a specified sync job |
| --submit <xml-file> | Submits a new job to the server |
| --sync-threads <thread-count> | Used in conjunction with --set-threads to set the number of sync threads to use for a job. |
| --set-threads <job-id> | Sets the number of sync threads on the server.  Requires --query-threads and/or  --sync-threads arguments |
| --status <job-id> | Queries the server for job status |
| --stop <job-id> | Terminates a specified sync job |
| --submit <xml-file> | Submits a new job to the server |
| --sync-threads <thread-count> | Used in conjunction with --set-threads to set the number of sync threads to use for a job. |

This tool is also useful to adjust the copy settings to get the maximum performance. You can modify, online, the number of threads and evaluate which thread count gives the fastest speed, for example.

And this is a typical output when querying the status of an ECS-Sync copy.

`java -jar ecs-sync-ctl-2.1.jar --status <job number>`

````
Job Time: 24h 30m 35s 148ms
Active Query Threads: 8
Active Sync Threads: 64
CPU Time: 300527390ms
CPU Usage: 67.6 %
Memory Usage: 9.3GB
Objects Completed: 14376550
Objects Expected: 20644739
Error Count: 0
Bytes Completed: 0B
Bytes Expected: 0B
Current BW (source): read: 0B/s write: 0B/s
Current BW (target): read: 0B/s write: 0B/s
Current Throughput: completed 33/s failed 0/s
Average BW: 0B/s
Average Throughput: 162.9/s
ETA: 10h 41m 10s 605ms

```

>**Note:** In order to enable performance monitoring for reads and writes when using the ECS-Sync CLI, you must include *--monitor-performance* in your copy command. 

When using the ECS-Sync UI it is also posible to modify a few of these parameters, like the number of threads. Please go through the *Advance options* in the UI to review these options.

### 3.3.2 -  ECS-Sync Copy parameters

This section explains how to configure and tune the most common parameters used in ECS-Sync copies.

**Task:** Please re-use the XML file you created in *Section 3.1 - NFS to ECS S3*, modifying source and target, so that you can tune the parameters explained in this section.

- Source: Create a File System according to *Apendix 2 - Generating a data set as a source for the NFS to ECS S3 copy*
- Target: Create a bucket called *bucketteargetnfs2*, owned by *ecssync_user2*

Run the copy job as many times as needed in order to play with the different options below. Be aware that the first copy will be a full copy while the following ones will be incremental (faster) copies.

#### *Number of threads*

Two different thread paremeters defined in ECS-Sync:

- *--query-threads:* Specifies the number of threads to use when querying for child objects.
- *--sync-threads:* Specifies the number of threads to use when syncing objects.

*Query threads* often match the *sync threads*, especially when migrating a huge number of directories from a File System.

However, the optimal number of threads will depend on your copy requirements. We recommend using a benchmarking tool to analyse and prevent bottlenecks before starting the migration. This helps to define the optimal configuration for the copy.

- We have found that, typically, 32 threads is a safe number, but it really depends on the number of directories (if copying from NFS) and average file size (or object size).
- For small objects, keep increasing threads until you hit a resource limit.  It is conceivable that you could run 64+ threads with 8 cores if the objects are small. If you hit a load limit on the sync VM, add more cores. ECS-Sync VM is scalable.

You can modify the number of query threads to 64, for example, using the `ecs-sync-ctl-xx.jar` file:

`java -jar ecs-sync-ctl-2.1.jar --set-threads <job number> --query-threads 64`

#### *Excluding directories*

When copying from a NFS source, it might be useful to exclude some folders from the source file system, for example, the snapshot directories. In that case you need to include the *--exclude-paths <path_to_exclude>* option either in the copy command or in the XMl file.  This uses regular expressions (for example, *\.snapshot*).

#### *Running several ECS-Sync sessions in the same ECS-Sync VM*

Using a single ECS-Sync VM, it is possible to run different ECS-Sync copy sessions. 

In order to achieve it, you need to add the *--rest-endpoint localhost:<free_port>* parameter in the copy command, specifying an available port in your VM.

For example:

`java -jar ecs-sync-3.0-beta.2.jar --rest-endpoint localhost:9202 --xml-config sample/directory-to-ecs-s3-test.xml`

You can check with `ps -ef | grep java` if there is another ECS-Sync instance running and using that port to avoid conflicts.

### 3.3.3 -  Incremental copies

By default, ECS-Sync performs incremental copies when re-running a copy after an initial copy. It calls the object HEAD first to see if the object exists before it is transferred.  If not, it PUTs the object.  

According to this algorithm, ECS-Sync does not synchronize deletes because it has no way of knowing that a delete happened in the source of the copy (it does not crawl the target).  This should be taken into consideration when migrating data with several iterations of incremental copies.

#### *Reprocess*

It is also possible to let ECS-Sync determine if an object must be re-copied or not based on the *mtime* and *object size*. When including *--reprocess* option in the copy command, ECS-Sync compares the *mtime* and the *size* in source and target, avoiding the copy of objects that are the same size and newer than the source.

#### *Force - full copy*

Finally, there is an option to force a full copy of the data, even if the object/file was copied before. In this case, the option *--force* should be added to the ECS-Sync copy command. It will copy/overwrite everything.

# 4 - ECS-Sync logs

It is higly recommended to track the output of the ECS-Sync copy command, since it will log all the errors and it will show the number of successful and failed files.

Log level can be set in the ECS-Sync copy command or in the ECS UI, and modified later on with *ecs-sync-ctl-xx.jar* tool or the *UI*. The log level can be DEBUG, INFO, WARN, ERROR, or FATAL. The default is ERROR.

There are several ways to track the copy output, either logging the output of the ECS-Sync copy command or using a database.

## 4.1 - Copy logs

When executing the ECS-Sync copy from the CLI, the CLI output will include errors, the number of files successfully copied, the number or files that failed, and, at the end, object keys that failed during the copy.

Depending on the number of errors, this output may be big. For this reason, *nohup* may be a good option to track the copy status & result:

`nohup java -jar ecs-sync-3.0-beta.2.jar --xml-config sample/directory-to-ecs-s3-test.xml`

Or, simply, redirecting the copy output to a file:

` java -jar ecs-sync-3.0-beta.2.jar --xml-config sample/directory-to-ecs-s3-test.xml 2>&1 | tee -a copy.log`

- Example of an output with no errors:

```
EcsSync v3.0-beta.2
2016-11-10 12:07:36 WARN  [main           ] RestServer: REST server listening at http://localhost:9200/
Transferred 10,744,882 bytes in 1 seconds (10,744,882 bytes/s)
Successful files: 6 (6/s) Failed Files: 0
Failed files: []
```

- Example of an output with errors, failed files at the end:

```
cat /target/nohup.out
EcsSync v2.1
2016-03-17 10:32:07 WARN  [main           ] RestServer: REST server listening at http://localhost:9200/
2016-03-17 10:32:15 WARN  [sync-pool-t-8  ] EcsSync: O--R object FileSyncObject(test6.txt) failed 1 time (queuing for retry): com.emc.object.s3.S3Exception: Internal Server Error
2016-03-17 10:32:15 WARN  [sync-pool-t-1  ] EcsSync: O--R object FileSyncObject(test19.txt) failed 1 time (queuing for retry): com.emc.object.s3.S3Exception: Internal Server Error
[…]
Transferred 25,470,485,194 bytes in 438 seconds (58,151,792 bytes/s)
Successful files: 968 (2.21/s) Failed Files: 8
Failed files: [S3SyncObject(PRUEBA/intocable.avi), S3SyncObject(SGDA/ACME/1482e0f0-a9c9-4a57-91fd-d79b0a070fdd.xml), S3SyncObject(SGDA/ACME/1e0c1c73-9082-4764-a4bc-f3a9721bcac4.xml), S3SyncObject(SGDA/ACME/033bbb30-a214-49ba-97cc-95b6ece9b04a.xml), S3SyncObject(SGDA/ACME/385e38a8-d405-4409-b9cb-b40ab2778bad.xml), S3SyncObject(SGDA/ACME/3e1dbeff-a6b0-4c09-ad46-44b0ab78b137.xml), S3SyncObject(SGDA/ACME/62c5e4c7-fcaa-4e0e-ab28-f8f94fdc97e3.xml), S3SyncObject(SGDA/ACME/5M_Altas_datosAbonados10GB.xml.xml.zip)]

```

## 4.2 - Database

It is also possible to use a Database to collect the status of the copy progress, including copied files/objects and failures.

In the ECS-Sync OVA there is an pre-installed MySQL/mariaDB database you can use. Therefore, the database itself can exist in a file (using the SQLite engine) or using MySQL/mariaDB.

You can either define the Database parameters in the XML file or in the ECS-Sync UI. Please take a look at the different options.

# Apendix 1 - Launching a ECS-Sync UI Docker Container in the vLab

This is a workaround to get the ECS-Sync UI available in the vLab. If you want to run the ECS-Sync in a customer site, deploying the ECS-Sync OVA is the preferred option.

In this case, we'll use an old ECS-Sync version since the ECS-Sync Ui is not available yet in 3.0.

- Pull the ECS-Sync Docker Container from GitHub:

`docker pull  djannot/ecs-sync-ui:2.1.2`

- Create a directory called */tmp/nfssource* in your Spark VM.

- Run the Docker Container

`docker run -it -v /tmp/nfssource:/tmp/nfssource --net=host djannot/ecs-sync-ui:2.1.2 bash`

Please, note that */tmp/nfssource* in the Docker container will be mapped to */tmp/nfssource* in your Spark VM, so that you can copy data from this directory.

- Run the ECS-Sync-UI:

`java -jar ecs-sync-ui-2.1.2.jar --server.address=192.168.1.30`

- Open a Web Browser and go to http://[http://192.168.1.30:8080/](http://192.168.1.30:8080/). After a few minutes the ECS-Sync UI will launch.
	- Default user/password: admin / ecs-sync


# Apendix 2 - Generating a data set as a source for the NFS to ECS S3 copy

In order to simulate a more realistic data set as source in the *filesystem_to_ecss3* copy, with tons of files and directories, please create a script file in */tmp/nfssource* and run it from there:

```
for a in {1..4};do
  mkdir $a
  for b in {1..10};do
    mkdir $a/$b
      for c in {1..10000};do
          echo "Create file $a/$b/file$c"
          echo $a$b$c > $a/$b/file$c
       done
   done
done

```

#References

- ECS-Sync documentation  https://community.emc.com/docs/DOC-38905
- For general questions on ECS-Sync and configuring your job file, please use the “ECS Sync Migrations” distribution list: ecs.sync.migrations@emc.com
- UI Service (OVA) Installation https://github.com/EMCECS/ecs-sync/wiki/UI---Service-(OVA)-Installation


