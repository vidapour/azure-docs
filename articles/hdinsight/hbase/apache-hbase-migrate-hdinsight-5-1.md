---
title: Migrate a HBase cluster to a HDInsight 5.1 - Azure HDInsight
description: Learn how to migrate Apache HBase clusters in Azure HDInsight to a HDInsight 5.1.
ms.service: azure-hdinsight
ms.topic: how-to
ms.custom: hdinsightactive
author: apurbasroy
ms.author: apsinhar
ms.reviewer: nijelsf
ms.date:  02/18/2025
---

# Migrate an Apache HBase cluster to a HDInsight 5.1

This article discusses how to update your Apache HBase cluster on Azure HDInsight to a newer version.

This article applies only if you use the same Azure Storage account for your source and destination clusters. To upgrade with a new or different Storage account for your destination cluster, see [Migrate Apache HBase to HDInsight 5.1 with a new Storage account](./apache-hbase-migrate-hdinsight-5-1-new-storage-account.md).

The downtime for upgrading may take more than 20 minutes. This downtime caused by the steps to flush all in-memory data, and wait for all procedure to complete and the time to configure and restart the services on the new cluster. Your results vary, depending on the number of nodes, amount of data, and other variables.

## Review Apache HBase compatibility

Before upgrading Apache HBase, ensure the HBase versions on the source and destination clusters are compatible. Review the HBase version compatibility matrix and release notes in the [HBase Reference Guide](https://hbase.apache.org/book.html#upgrading) to make sure your application is compatible with the new version.

Here's an example compatibility matrix. **Y** indicates compatibility and **N** indicates a potential incompatibility:

| Compatibility type | Major version| Minor version | Patch |
| --- | --- | --- | --- |
| Client-Server wire compatibility | N | Y | Y |
| Server-Server compatibility | N | Y | Y |
| File format compatibility | N | Y | Y |
| Client API compatibility | N | Y | Y |
| Client binary compatibility | N | N | Y |
| **Server-side limited API compatibility** |  |  |  |
| Stable | N | Y | Y |
| Evolving | N | N | Y |
| Unstable | N | N | N |
| Dependency compatibility | N | Y | Y |
| Operational compatibility | N | N | Y |

For more information about HDInsight versions and compatibility, see [Azure HDInsight versions](../hdinsight-component-versioning.md).

## Apache HBase cluster migration overview

To upgrade your Apache HBase cluster on Azure HDInsight, complete the following basic steps. For detailed instructions, see the detailed steps and commands, or use the scripts from the section [Migrate HBase using scripts](#migrate-hbase-using-scripts) for automated migration.

Prepare the source cluster:
1. Stop data ingestion.
1. Check cluster health
1. Stop replication if needed
1. Flush ``memstore``data.
1. Stop HBase.
1. For clusters with accelerated writes, back up the Write Ahead Log (WAL) directory.

Prepare the destination cluster:
1. Create the destination cluster.
1. Stop HBase from Ambari.
1. Update `fs.defaultFS` in HDFS service configs to refer to the original source cluster container.
1. For clusters with accelerated writes, update `hbase.rootdir` in HBase service configs to refer to the original source cluster container.
1. Clean Zookeeper data.

Complete the migration:
1. Clean and migrate the WAL.
1. Copy apps from the destination cluster's default container to the original source container.
1. Start all services from the Ambari destination cluster.
1. Verify HBase.
1. Delete the source cluster.

## Detailed migration steps and commands

Use these detailed steps and commands to migrate your Apache HBase cluster.

### Prepare the source cluster

1. Stop ingestion to the source HBase cluster.

1. Check HBase hbck to verify cluster health
   
   1. Verify HBCK Report page on HBase UI.  Healthy cluster doesn't show any inconsistencies
   :::image type="content" source="./media/apache-hbase-migrate-new-version/verify-hbck-report.png" alt-text="Screenshot showing how to verify HBCK report." lightbox="./media/apache-hbase-migrate-new-version/verify-hbck-report.png":::
   1. If any inconsistencies exist, fix inconsistencies using [hbase hbck2](/azure/hdinsight/hbase/how-to-use-hbck2-tool/)

1. Note down number of regions in online at source cluster, so that the number can be referred at destination cluster after the migration. 
   :::image type="content" source="./media/apache-hbase-migrate-new-version/total-number-of-regions.png" alt-text="Screenshot showing total number of regions." lightbox="./media/apache-hbase-migrate-new-version/total-number-of-regions.png":::

1. If replication enabled on the cluster, stop and reenable the replication on destination cluster after migration. For more information, see [HBase replication guide](/azure/hdinsight/hbase/apache-hbase-replication/)  

1. Flush the source HBase cluster you're upgrading.
   
   HBase writes incoming data to an in-memory store called a *`memstore`*. After the ``memstore``reaches a certain size, HBase flushes it to disk for long-term storage in the cluster's storage account. Deleting the source cluster after an upgrade also deletes any data in the `memstore`s. To retain the data, manually flush each table's ``memstore``to disk before upgrading.
   
   You can flush the ``memstore``data by running the [flush_all_tables.sh](https://github.com/Azure/hbase-utils/blob/master/scripts/flush_all_tables.sh) script from the [Azure hbase-utils GitHub repository](https://github.com/Azure/hbase-utils/).
   
   You can also flush ``memstore``data by running the following HBase shell command from the HDInsight cluster:
   
   ```bash
   hbase shell
   flush "<table-name>"
   ```
1. Wait for 15 mins and verify that all the procedures are completed, and masterProcWal files doesn't  have any pending procedures.

    1. Verify the Procedures page to confirm that there are no pending procedures.
    
        :::image type="content" source="./media/apache-hbase-migrate-new-version/verify-master-process.png" alt-text="Screenshot showing how to verify master process." lightbox="./media/apache-hbase-migrate-new-version/verify-master-process.png"::: 

1. STOP HBase

   1. Sign in to [Apache Ambari](https://ambari.apache.org/) on the source cluster with `https://<OLDCLUSTERNAME>.azurehdinsight.net`
   1. Turn on maintenance mode for HBase.
   1. Stop HBase Masters only first. First stop standby masters, in last stop Active HBase master.
   
       :::image type="content" source="./media/apache-hbase-migrate-new-version/stop-master-services.png" alt-text="Screenshot showing how to stop master services." lightbox="./media/apache-hbase-migrate-new-version/stop-master-services.png":::
   
   1. Stop the HBase service, it stops remaining servers.
   
   > [!NOTE]
   > HBase 2.4.11 doesn't support some of the old Procedures. 
   >
   >For more information on connecting to and using Ambari, see [Manage HDInsight clusters by using the Ambari Web UI](../hdinsight-hadoop-manage-ambari.md).
   >
   >Stopping HBase in the previous steps to avoid creating new master proc WALs.

1. If your source HBase cluster doesn't have the [Accelerated Writes](apache-hbase-accelerated-writes.md) feature, skip this step. For source HBase clusters with Accelerated Writes, back up the WAL directory under HDFS by running the following commands from an SSH session on any of the Zookeeper nodes or worker nodes of the source cluster.
   
   ```bash
   hdfs dfs -mkdir /hbase-wal-backup
   hdfs dfs -cp hdfs://mycluster/hbasewal /hbase-wal-backup
   ```
   
### Prepare the destination cluster

1. In the Azure portal, [set up a new destination HDInsight cluster](../hdinsight-hadoop-provision-linux-clusters.md) using the same storage account as the source cluster, but with a different container name:

   
1. Sign in to [Apache Ambari](https://ambari.apache.org/) on the new cluster at `https://<NEWCLUSTERNAME>.azurehdinsight.net`, and stop the HBase services.
   
1. Under **Services** > **HDFS** > **Configs** > **Advanced** > **Advanced core-site**, change the `fs.defaultFS` HDFS setting to point to the original source cluster container name. For example, the setting in the following screenshot should be changed to `wasbs://hbase-upgrade-old-2021-03-22`.
   
   :::image type="content" source="./media/apache-hbase-migrate-new-version/hdfs-advanced-settings.png" alt-text="Screenshot shows in Ambari, select Services > HDFS > Configs > Advanced > Advanced core-site and change the container name." lightbox="./media/apache-hbase-migrate-new-version/hdfs-advanced-settings.png":::
   
1. If your destination cluster has the Accelerated Writes feature, change the `hbase.rootdir` path to point to the original source cluster container name. For example, the following path should be changed to `hbase-upgrade-old-2021-03-22`. If your cluster doesn't have Accelerated Writes, skip this step.
   
   :::image type="content" source="./media/apache-hbase-migrate-new-version/change-container-name-for-hbase-rootdir.png" alt-text="Screenshot shows in Ambari, change the container name for the HBase rootdir." border="true" lightbox="./media/apache-hbase-migrate-new-version/change-container-name-for-hbase-rootdir.png":::
   
1. Clean the Zookeeper data on the destination cluster by running the following commands in any Zookeeper node or worker node:
   
   ```bash
   hbase zkcli
   deleteall /hbase-unsecure
   quit
   ```
   > [!NOTE]
   > In some versions of Apache ZooKeeper, `rmr` is used instead of `deleteall`.

### Clean and migrate WAL

Run the following commands, depending on your source HDInsight version and whether the source and destination clusters have Accelerated Writes.

1. The destination cluster is always HDInsight version 4.0, since HDInsight 3.6 is in Basic support and isn't recommended for new clusters.
1. The HDFS copy command is `hdfs dfs <copy properties starting with -D> -cp <source> <destination> # Serial execution`.

> [!NOTE]  
> - The `<source-container-fullpath>` for storage type WASB is `wasbs://<source-container-name>@<storageaccountname>.blob.core.windows.net`.
> - The `<source-container-fullpath>` for storage type Azure Data Lake Storage Gen2 is `abfs://<source-container-name>@<storageaccountname>.dfs.core.windows.net`.

- [The source cluster is HDInsight 4.0 with Accelerated Writes, and the destination cluster has Accelerated Writes](#the-source-cluster-is-hdinsight-40-with-accelerated-writes-and-the-destination-cluster-has-accelerated-writes).
- [The source cluster is HDInsight 4.0 without Accelerated Writes, and the destination cluster has Accelerated Writes](#the-source-cluster-is-hdinsight-40-without-accelerated-writes-and-the-destination-cluster-has-accelerated-writes).
- [The source cluster is HDInsight 4.0 without Accelerated Writes, and the destination cluster doesn't have Accelerated Writes](#the-source-cluster-is-hdinsight-40-without-accelerated-writes-and-the-destination-cluster-doesnt-have-accelerated-writes).

#### The source cluster is HDInsight 4.0 with Accelerated Writes, and the destination cluster has Accelerated Writes

Clean the WAL FS data for the destination cluster, and copy the WAL directory from the source cluster into the destination cluster's HDFS. Copy the directory by running the following commands in any Zookeeper node or worker node on the destination cluster:

```bash   
sudo -u hbase hdfs dfs -rm -r hdfs://mycluster/hbasewal
sudo -u hbase hdfs dfs -cp <source-container-fullpath>/hbase-wal-backup/hbasewal hdfs://mycluster/
```

#### The source cluster is HDInsight 4.0 without Accelerated Writes, and the destination cluster has Accelerated Writes

Clean the WAL FS data for the destination cluster, and copy the WAL directory from the source cluster into the destination cluster's HDFS. Copy the directory by running the following commands in any Zookeeper node or worker node on the destination cluster:

```bash
sudo -u hbase hdfs dfs -rm -r hdfs://mycluster/hbasewal
sudo -u hbase hdfs dfs -cp <source-container-fullpath>/hbase-wals/* hdfs://mycluster/hbasewal
   ```
#### The source cluster is HDInsight 4.0 without Accelerated Writes, and the destination cluster doesn't have Accelerated Writes

Clean the WAL FS data for the destination cluster, and copy the source cluster WAL directory into the destination cluster's HDFS. To copy the directory, run the following commands in any Zookeeper node or worker node on the destination cluster:

```bash
sudo -u hbase hdfs dfs -rm -r /hbase-wals/*
sudo -u hbase hdfs dfs -Dfs.azure.page.blob.dir="/hbase-wals" -cp <source-container-fullpath>/hbase-wals /
```
### Complete the migration

1. Using the `sudo -u hdfs` user context, copy the folder `/hdp/apps/<new-version-name>` and its contents from the `<destination-container-fullpath>` to the `/hdp/apps` folder under `<source-container-fullpath>`. You can copy the folder by running the following commands on the destination cluster:
   
   ```bash   
   sudo -u hdfs hdfs dfs -cp /hdp/apps/<hdi-version> <source-container-fullpath>/hdp/apps
   ```
   
   For example:
   ```bash
   sudo -u hdfs hdfs dfs -cp /hdp/apps/4.1.3.6 wasbs://hbase-upgrade-old-2021-03-22@hbaseupgrade.blob.core.windows.net/hdp/apps
   ```
   
1. On the destination cluster, save your changes, and restart all required services as Ambari indicates.
   
1. Point your application to the destination cluster.
   
   > [!NOTE]  
   > The static DNS name for your application changes when you upgrade. Rather than hard-coding this DNS name, you can configure a CNAME in your domain name's DNS settings that points to the cluster's name. Another option is to use a configuration file for your application that you can update without redeploying.
   
1. Start the ingestion.
   
1. Verify HBase consistency and simple Data Definition Language (DDL) and Data Manipulation Language (DML) operations.
   
1. If the destination cluster is satisfactory, delete the source cluster.

## Migrate HBase using scripts

1. Note down number of regions in online at source cluster, so that the number can be referred at destination cluster after the migration.
   :::image type="content" source="./media/apache-hbase-migrate-new-version/total-number-of-regions.png" alt-text="Screenshot showing count of number of regions." lightbox="./media/apache-hbase-migrate-new-version/total-number-of-regions.png":::

1. Flush the source HBase cluster you're upgrading.
   
   HBase writes incoming data to an in-memory store called a *`memstore`*. After the ``memstore``reaches a certain size, HBase flushes it to disk for long-term storage in the cluster's storage account. Deleting the source cluster after an upgrade also deletes any data in the `memstore`s. To retain the data, manually flush each table's ``memstore``to disk before upgrading.
   
   You can flush the ``memstore``data by running the [flush_all_tables.sh](https://github.com/Azure/hbase-utils/blob/master/scripts/flush_all_tables.sh) script from the [Azure hbase-utils GitHub repository](https://github.com/Azure/hbase-utils/).
   
   You can also flush ``memstore``data by running the following HBase shell command from the HDInsight cluster:
   
   ```bash
   hbase shell
   flush "<table-name>"
   ```
1. Wait for 15 mins and verify that all the procedures are completed, and masterProcWal files doesn't  have any pending procedures.

    1. Verify the Procedures page to confirm that there are no pending procedures.
    
        :::image type="content" source="./media/apache-hbase-migrate-new-version/verify-master-process.png" alt-text="Screenshot showing how to verify master process." lightbox="./media/apache-hbase-migrate-new-version/verify-master-process.png"::: 


1. Execute the script [migrate-to-HDI5.1-hbase-source.sh ](https://github.com/Azure/hbase-utils/blob/master/scripts/migrate-to-HDI5.1-hbase-source.sh) on the source cluster and [migrate-hbase-dest.sh](https://github.com/Azure/hbase-utils/blob/master/scripts/migrate-hbase-dest.sh) on the destination cluster. Use the following instructions to execute these scripts.
   > [!NOTE]  
   > These scripts don't copy the HBase old WALs as part of the migration; therefore, the scripts are not to be used on clusters that have either HBase Backup or Replication feature enabled.

2. On source cluster
   ```bash
   sudo bash migrate-to-HDI5.1-hbase-source.sh 
   ```
   
3. On destination cluster
   ```bash
   sudo bash migrate-hbase-dest.sh  -f <src_default_Fs>
   ```
   
Mandatory argument for the above command:
   
```
   -f, --src-fs
   The fs.defaultFS of the source cluster
   For example:
   -f wasb://anynamehbase0316encoder-2021-03-17t01-07-55-935z@anynamehbase0hdistorage.blob.core.windows.net
```


## Troubleshooting

### Use case 1: 
If HBase masters and region servers up and regions stuck in transition or only one region, for example, `hbase:meta` region is assigned. Waiting for other regions to assign

**Solution:**

1. ssh into any ZooKeeper node of original cluster and run `kinit -k -t /etc/security/keytabs/hbase.service.keytab hbase/<zk FQDN>` if this is ESP cluster
1. Run `echo "scan '`hbase:meta`'" | hbase shell > meta.out` to read the `hbase:meta` into a file
1. Run `grep "info:sn" meta.out | awk '{print $4}' | sort | uniq`  to get all RS instance names where the regions were present in old cluster. Output should be like `value=<wn FQDN>,16020,........`
1. Create a dummy WAL dir with that `wn` value

   If the cluster is accelerated write cluster 
   ```
   hdfs dfs -mkdir hdfs://mycluster/hbasewal/WALs/<wn FQDN>,16020,.........
   ```
   If the cluster is nonaccelarated Write cluster 
   ```
   hdfs dfs -mkdir /hbase-wals/WALs/<wn FQDN>,16020,.........
   ```
1. Restart active `Hmaster`

## Next steps

To learn more about [Apache HBase](https://hbase.apache.org/) and upgrading HDInsight clusters, see the following articles:

- [Upgrade an HDInsight cluster to a newer version](../hdinsight-upgrade-cluster.md)
- [Monitor and manage Azure HDInsight using the Apache Ambari Web UI](../hdinsight-hadoop-manage-ambari.md)
- [Azure HDInsight versions](../hdinsight-component-versioning.md)
- [Optimize Apache HBase](../optimize-hbase-ambari.md)
