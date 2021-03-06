<!--
{% comment %}
Licensed to the Apache Software Foundation (ASF) under one or more
contributor license agreements.  See the NOTICE file distributed with
this work for additional information regarding copyright ownership.
The ASF licenses this file to you under the Apache License, Version 2.0
(the "License"); you may not use this file except in compliance with
the License.  You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
{% endcomment %}
-->

## Ambari Infra Solr Client

CLI helper tool(s) for Ambari Infra Solr.

### Post Ambari Server Upgrade (Ambari 2.7.x)

Ambari Infra Solr uses Solr 7 from Ambari 2.7.0, therefore it is required migrate Solr 5 index (Ambari Infra 2.6.x), if you want to keep your old data. (otherwise backup part can be skipped)

#### Contents:
- [I. Upgrade Ambari Infra Solr Clients](#i.-upgrade-ambari-infra-solr-client)
- [II. Gather required Ambari and Solr parameters](#0-gather-params)
- [III. Backup Solr Collections](#ii.-backup-collections-(ambari-2.6.x-to-ambari-2.7.x))
    - a.) If you have Ranger Ambari service with Solr audits:
        - [1. Backup Ranger collection](#ii/1.-backup-ranger-collection)
        - [2. Backup Ranger configs on Solr ZNode](#ii/2.-backup-ranger-configs-on-solr-znode)
        - [3. Delete Ranger collection](#ii/3.-delete-ranger-collection)
        - [4. Upgrade Ranger Solr schema](#ii/4.-upgrade-ranger-solr-schema)
    - b.) If you have Atlas Ambari service:
        - [5. Backup Atlas collections](#ii/5.-backup-atlas-collections)
        - [6. Delete Atlas collections](#ii/6.-delete-atlas-collections)
    - c.) If you have Log Search Ambari service:
        - [7. Delete Log Search collections](#ii/7.-delete-log-search-collections)
        - [8. Delete Log Search Solr configs](#ii/8.-delete-log-search-solr-configs)
- [IV. Upgrade Ambari Infra Solr package](#iii.-upgrade-infra-solr-packages)
- [V. Re-create Solr Collections](#iv.-re-create-collections)
- [VI. Migrate Solr Collections](#v.-migrate-solr-collections)
    - a.) If you have Ranger Ambari service with Solr audits:
        - [1. Migrate Ranger Solr collection](#v/1.-migrate-ranger-collections)
    - b.) If you have Atlas Ambari service:
        - [2. Migrate Atlas Solr collections](#v/2.-migrate-atlas-collections)
- [VII. Restore Solr Collections](#vi.-restore-collections)
    - a.) If you have Ranger Ambari service with Solr audits:
        - [1. Restore old Ranger collection](#vi/1.-restore-old-ranger-collection)
    - b.) If you have Atlas Ambari service:
        - [4. Restore old Atlas collections](#vi/4.-restore-old-atlas-collections)
- [VIII. Restart Solr Instances](#vii.-restart-infra-solr-instances)
- [IX. Transport old data to new collections](#viii.-transport-old-data-to-new-collections)
    - a.) If you have Ranger Ambari service with Solr audits:
        - [1. Transport old data to Ranger collection](#viii/1.-transport-old-data-to-ranger-collection)
    - b.) If you have Atlas Ambari service:
        - [2. Transport old data to Atlas collections](#viii/2.-transport-old-data-to-atlas-collections)
- [Happy Path](#happy-path)
- [APPENDIX](#appendix)

### <a id="i.-upgrade-ambari-infra-solr-client">I. Upgrade Ambari Infra Solr Client</a>

First make sure `ambari-infra-solr-client` is the latest. (If its before 2.7.x) It will contain the migrationHelper.py script at `/usr/lib/ambari-infra-solr-client` location.
Also make sure you won't upgrade `ambari-infra-solr` until the migration has not done. (all of this should happen after `ambari-server` upgrade, also make sure to not restart `INFRA_SOLR` instances). 
Optionally, you can stop ranger plugins at this point.

### <a id="0-gather-params">II. Gather required Ambari and Solr parameters</a>

At the start, it is required to create a proper configuration input for the migration helper script. That can be done with [/usr/lib/ambari-infra-solr-client/migrationConfigGenerator.py](#migration-config-generator) script. Choose one of the Solr server host, and ssh there and run (with proper ambari-server configurations as flags):

```bash
# use a sudoer user for running the script !!
CONFIG_INI_LOCATION=ambari_solr_migration.ini # output of the script with required parameters for migrationHelper.py
# note 1: use -s if ambari-server uses https
# note 2: use --shared-driver if the backup location is shared for different hosts
# note 3: use --hdfs-base-path if the index data is located on hdfs (or --ranger-hdfs-base-path if only ranger collection is located there), e.g.: /user/infra-solr
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationConfigGenerator.py --ini-file $CONFIG_INI_LOCATION --host c7401.ambari.apache.org --port 8080 --cluster cl1 --username admin --password admin --backup-base-path=/my/path --java-home /usr/jdk64/jdk1.8.0_112
```

The generated config file output could be something like that:
```ini
[ambari_server]
host = c7401.ambari.apache.org
port = 8080
cluster = cl1
protocol = http
username = admin
password = admin

[local]
java_home = /usr/jdk64/jdk1.8.0_112/
hostname = c7402.ambari.apache.org
shared_drive = false

[cluster]
kerberos_enabled = true

[infra_solr]
protocol = http
urls = http://c7402.ambari.apache.org:8886/solr,http://c7403.ambari.apache.org:8886/solr
zk_connect_string = c7401.ambari.apache.org:2181
znode = /infra-solr
user = infra-solr
keytab = /etc/security/keytabs/ambari-infra-solr.service.keytab
principal = infra-solr/c7402.ambari.apache.org
zk_principal_user = zookeeper

[ranger_collection]
enabled = true
ranger_config_set_name = ranger_audits
ranger_collection_name = ranger_audits
ranger_collection_shards = 2
ranger_collection_max_shards_per_node = 4
backup_ranger_config_set_name = old_ranger_audits
backup_ranger_collection_name = old_ranger_audits
backup_path = /my/path/ranger

[atlas_collections]
enabled = true
config_set = atlas_configs
fulltext_index_name = fulltext_index
fulltext_index_shards = 2
fulltext_index_max_shards_per_node = 4
edge_index_name = edge_index
edge_index_shards = 2
edge_index_max_shards_per_node = 4
vertex_index_name = vertex_index
vertex_index_shards = 2
vertex_index_max_shards_per_node = 4
backup_fulltext_index_name = old_fulltext_index
backup_edge_index_name = old_edge_index
backup_vertex_index_name = old_vertex_index
backup_path = /my/path/atlas

[logsearch_collections]
enabled = true
hadoop_logs_collection_name = hadoop_logs
audit_logs_collection_name = audit_logs
history_collection_name = history
```

After the file has created successfully by the script, review the configuration (e.g.: if 1 of the Solr is not up yet, and you do not want to use its REST API for operations, you can remove its url from the urls of infra_solr section or you can change backup locations for different collections etc.)

### <a id="ii.-backup-collections-(ambari-2.6.x-to-ambari-2.7.x)">III. Backup collections (Ambari 2.6.x to Ambari 2.7.x)</a>

Before you start to upgrade process check the Solr instances are running and also make sure you have stable shards (at least one core is up and running) and will have enough space on the disks to store Solr backup data. (you will need at least that many as your index size per host). The backup process contains a few steps: backup ranger configs on znode, backup collections, delete Log Search znodes, then upgrade `managed-schema` znode for Ranger. 
These tasks can be done with 1 [migrationHelper.py](#solr-migration-helper-script) command:

```bash
# use a sudoer user for running the script !!
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action backup-and-cleanup
```

If the script finished successfully and everything looks green on Ambari UI as well, you can go ahead with [Infra Solr package upgrade](#iii.-upgrade-infra-solr-packages). Otherwise (or if you want to go step by step instead of the command above) you have to option to run tasks step by step (or manually as well). Those tasks are found in the next sections.

#### <a id="ii/1.-backup-ranger-collection">III/1. Backup Ranger collection</a>

The [migrationHelper.py](#solr-migration-helper-script) script can be used to backup only Ranger collection (use `-s` option to filter on services)

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action backup -s RANGER
```

Also you can do the backup manually on every Solr node, by using [backup API of Solr](https://lucene.apache.org/solr/guide/6_6/making-and-restoring-backups.html). (use against core names, not collection name, it works as expected only if you have 1 shard on every node)

Example:
```bash

su infra-solr
SOLR_URL=... # actual solr host url, example: http://c6401.ambari.apache.org:8886/solr
# collection parameters
BACKUP_PATH=... # backup location, e.g.: /tmp/ranger-backup

# RUN THIS FOR EVERY CORE ON SPECIFIC HOSTS !!!
BACKUP_CORE=... # specific core on a host
BACKUP_CORE_NAME=... # core names for backup -> <backup_location>/
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)
mkdir -p $BACKUP_PATH

curl --negotiate -k -u : "$SOLR_URL/$BACKUP_CORE/replication?command=BACKUP&location=$BACKUP_PATH&name=$BACKUP_CORE_NAME"
```

(help: [get core names](#get-core-/-shard-names-with-hosts))

#### <a id="ii/2.-backup-ranger-configs-on-solr-znode">III/2. Backup Ranger configs on Solr ZNode</a>

Next you can copy `ranger_audits` configs to a different znode, in order to keep the old schema.

```bash
export JAVA_HOME=/usr/jdk64/1.8.0_112 # or other jdk8 location
export ZK_CONN_STR=... # without znode, e.g.: myhost1:2181,myhost2:2181,myhost3:2181
# note 1: --transfer-mode copyToLocal or --transfer-mode copyFromLocal can be used if you want to use the local filesystem
# note 2: use --jaas-file option only if the cluster is kerberized
infra-solr-cloud-cli --transfer-znode -z $ZK_CONN_STR --jaas-file /etc/ambari-infra-solr/conf/infra_solr_jaas.conf --copy-src /infra-solr/configs/ranger_audits --copy-dest /infra-solr/configs/old_ranger_audits
```

#### <a id="ii/3.-delete-ranger-collection">III/3. Delete Ranger collection</a>

At this point you can delete the actual Ranger collection with this command:

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action delete-collections -s RANGER
```

Or do it manually by the Solr API:

```bash
su infra-solr # infra-solr user - if you have a custom one, use that
SOLR_URL=... # example: http://c6401.ambari.apache.org:8886/solr
COLLECTION_NAME=ranger_audits

# use kinit and --negotiate option for curl only if the cluster is kerberized
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)

curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=DELETE&name=$COLLECTION_NAME"
```

#### <a id="ii/4.-upgrade-ranger-solr-schema">III/4. Upgrade Ranger Solr schema</a>

Before creating the new Ranger collection, it is required to upgrade `managed-schema` configs.

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action cleanup-znodes -s RANGER
```

It can be done manually by `infra-solr-cloud-cli` as well:

```bash
sudo -u infra-solr -i

# If kerberos enabled
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)

## BACKUP OLD CONFIG
export JAVA_HOME=/usr/jdk64/1.8.0_112 # or other jdk8 location
export ZK_CONN_STR=... # without znode, e.g.: myhost1:2181,myhost2:2181,myhost3:2181
# note: --transfer-mode copyToLocal or --transfer-mode copyFromLocal can be used if you want to use the local filesystem
infra-solr-cloud-cli --transfer-znode -z $ZK_CONN_STR --jaas-file /etc/ambari-infra-solr/conf/infra_solr_jaas.conf --copy-src /infra-solr/configs/ranger_audits --copy-dest /infra-solr/configs/old_ranger_audits
## UPLOAD NEW SCHEMA
# Setup env for zkcli.sh
source /etc/ambari-infra-solr/conf/infra-solr-env.sh
# Run that command only if kerberos is enabled.
export SOLR_ZK_CREDS_AND_ACLS="${SOLR_AUTHENTICATION_OPTS}"

# Upload the new schema
/usr/lib/ambari-infra-solr/server/scripts/cloud-scripts/zkcli.sh --zkhost "${ZK_HOST}" -cmd putfile /configs/ranger_audits/managed-schema /usr/lib/ambari-infra-solr-client/migrate/managed-schema
```

#### <a id="ii/5.-backup-atlas-collections">III/5. Backup Atlas collections</a>

Atlas has 3 collections: fulltext_index, edge_index, vertex_index.
You will need to do similar steps that you did for Ranger, only difference is you will need to filter ATLAS service.

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action backup -s ATLAS
```

Also you can do the backup manually on every Solr node, by using [backup API of Solr](https://lucene.apache.org/solr/guide/6_6/making-and-restoring-backups.html). (use against core names, not collection name, it works as expected only if you have 1 shard on every node)

Example:
```bash

su infra-solr
SOLR_URL=... # actual solr host url, example: http://c6401.ambari.apache.org:8886/solr
# collection parameters
BACKUP_PATH=... # backup location, e.g.: /tmp/fulltext_index_backup

# RUN THIS FOR EVERY CORE ON SPECIFIC HOSTS !!!
BACKUP_CORE=... # specific core on a host
BACKUP_CORE_NAME=... # core names for backup -> <backup_location>/
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)
mkdir -p $BACKUP_PATH

curl --negotiate -k -u : "$SOLR_URL/$BACKUP_CORE/replication?command=BACKUP&location=$BACKUP_PATH&name=$BACKUP_CORE_NAME"
```
(help: [get core names](#get-core-/-shard-names-with-hosts))

#### <a id="ii/6.-delete-atlas-collections">III/6. Delete Atlas collections</a>

Next step for Atlas is to delete all 3 old collections. It can be done by `delete-collections` action with ATLAS filter.

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action delete-collections -s ATLAS
```

Or manually run DELETE operation with 3 Solr API call on all 3 Atlas collections:

```bash
su infra-solr # infra-solr user - if you have a custom one, use that
SOLR_URL=... # example: http://c6401.ambari.apache.org:8886/solr

# use kinit and --negotiate option for curl only if the cluster is kerberized
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)

COLLECTION_NAME=fulltext_index
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=DELETE&name=$COLLECTION_NAME"
COLLECTION_NAME=edge_index
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=DELETE&name=$COLLECTION_NAME"
COLLECTION_NAME=vertex_index
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=DELETE&name=$COLLECTION_NAME"
```

#### <a id="ii/7.-delete-log-search-collections">III/7. Delete Log Search collections</a>

For Log Search, it is a must to delete all the old collections. Can be done similar way as for Ranger or Atlas:

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action delete-collections -s LOGSEARCH
```
Or manually run Solr API DELETE commands here as well:
```bash
su infra-solr # infra-solr user - if you have a custom one, use that
SOLR_URL=... # example: http://c6401.ambari.apache.org:8886/solr

# use kinit and --negotiate option for curl only if the cluster is kerberized
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)

COLLECTION_NAME=hadoop_logs
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=DELETE&name=$COLLECTION_NAME"
COLLECTION_NAME=audit_logs
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=DELETE&name=$COLLECTION_NAME"
COLLECTION_NAME=history
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=DELETE&name=$COLLECTION_NAME"
```

#### <a id="ii/8.-delete-log-search-solr-configs">III/8. Delete Log Search Solr configs</a>

Log Search configs are changed a lot between Ambari 2.6.x and Ambari 2.7.x, so it is required to delete those as well. (configs will be regenerated during Log Search startup)
```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action cleanup-znodes -s LOGSEARCH
```
You can delete the znodes by zookeeper-client as well:

```bash
su infra-solr # infra-solr user - if you have a custom one, use that
# ZOOKEEPER CONNECTION STRING from zookeeper servers
export ZK_CONN_STR=... # without znode,e.g.: myhost1:2181,myhost2:2181,myhost3:2181

kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)

zookeeper-client -server $ZK_CONN_STR rmr /infra-solr/configs/hadoop_logs
zookeeper-client -server $ZK_CONN_STR rmr /infra-solr/configs/audit_logs
zookeeper-client -server $ZK_CONN_STR rmr /infra-solr/configs/history
```

### <a id="iii.-upgrade-infra-solr-packages">IV. Upgrade Infra Solr packages</a>

At this step, you will need to upgrade `ambari-infra-solr` packages. (also make sure ambari-logsearch* packages are upgraded as well)

Example (for CentOS):
```bash
yum upgrade -y ambari-infra-solr
```

### <a id="iv.-re-create-collections">V. Re-create collections</a>

Restart Ranger Admin / Atlas / Log Search Ambari service, as the collections were deleted before, during startup, new collections will be created (as a Solr 7 collection).
At this point you can stop, and do the migration / restore later (until you will have the backup), and go ahead with e.g. HDP upgrade. (migration part can take long - 1GB/min.)

### <a id="v.-migrate-solr-collections">VI. Migrate Solr Collections</a>

From this point, you can migrate your old index in the background. On every hosts, where there is a backup located, you can run luce index migration tool (packaged with ambari-infra-solr-client).. For lucene index migration, [migrationHelper.py](#solr-migration-helper-script) can be used, or `/usr/lib/ambari-infra-solr-client/solrIndexHelper.sh` directly. That script uses [IndexMigrationTool](#https://lucene.apache.org/solr/guide/7_3/indexupgrader-tool.html)
The whole migration can be done with execuing 1 command;
```bash
# use a sudoer user for running the script !!
# you can use this command with nohup in the background, like: `nohup <command> > nohup2.out&`, as migration can take so much time (~1GB/min)
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action migrate
```
If the script finished successfully and everything looks green on Ambari UI as well, you can go ahead with [Restore collections](#vi.-restore-collections). Otherwise (or if you want to go step by step instead of the command above) you have to option to run tasks step by step (or manually as well). Those tasks are found in the next sections.

#### <a id="v/1.-migrate-ranger-collections">VI/1. Migrate Ranger collections</a>

Migration for `ranger_audits` collection (cores):

```bash
# by efault, you will mirate to Lucene 6.6.2, if you want to migrate again to Solr 7 (not requred), you can use --version 7.3.1 flag
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action migrate -s RANGER
```

Or you can run commands manually on nodes where your backups are located:
```bash

export JAVA_HOME=/usr/jdk64/1.8.0_112

# if /tmp/ranger-backup is your backup location
infra-lucene-index-tool upgrade-index -d /tmp/ranger-backup -f -b -g

# with 'infra-lucene-index-tool help' command you can checkout the command line options
```

By default, the tool will migrate from lucene version 5 to lucene version 6.6.2. (that's ok for Solr 7) If you want a lucene 7 index, you will need to re-run the migration tool command with `-v 7.3.1` option.

#### <a id="v/2.-migrate-atlas-collections">VI/2. Migrate Atlas collections</a>

As Atlas has 3 collections, you will need similar steps that is required for Ranger, just for all 3 collections.
(fulltext_index, edge_index, vertex_index)

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action migrate -s ATLAS
```

Or you can run commands manually on nodes where your backups are located:
```bash

export JAVA_HOME=/usr/jdk64/1.8.0_112

# if /tmp/fulltext_index_backup is your backup location
infra-lucene-index-tool upgrade-index -d /tmp/fulltext_index_backup -f -b -g

# with 'infra-lucene-index-tool help' command you can checkout the command line options
```

By default, the tool will migrate from lucene version 5 to lucene version 6.6.2. (that's ok for Solr 7) If you want a lucene 7 index, you will need to re-run the migration tool command with `-v 7.3.1` option.

### <a id="vi.-restore-collections">VII. Restore Collections</a>

For restoring the old collections, first you will need to create them. As those collections could be not listed in the security.json of Infra Solr, you can get 403 errors if you will try to access those collections later, for that time until you are doing the restoring + transport solr data to another collections, you can [trun off](#turn-off-infra-solr-authorization) the Solr authorization plugin.

The collection creation and restore part can be done with 1 command:

```bash
# use a sudoer user for running the script !!
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action restore
```

If the script finished successfully and everything looks green on Ambari UI as well, you can go ahead with [Restart Solr Instances](#viii.-restart-infra-solr-instances). Otherwise (or if you want to go step by step instead of the command above) you have to option to run tasks step by step (or manually as well). Those tasks are found in the next sections.

#### <a id="vi/1.-restore-old-ranger-collection">VII/1. Restore Old Ranger collection</a>

After lucene data migration is finished, you can restore your replicas on every hosts where you have the backups. But we need to restore the old data to a new collection, so first you will need to create that: (on a host where you have an installed Infra Solr component). For Ranger, use old_ranger_audits config set that you backup up during Solr schema config upgrade step. (set this as CONFIG_NAME), to make that collection to work with Solr 7, you need to copy your solrconfig.xml as well.
That can be done with executing the following command:
```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action restore -s RANGER
```

Or you can manually create a collection for restoring the backup (`old_ranger_audits`)

```bash
su infra-solr # infra-solr user - if you have a custom one, use that
SOLR_URL=... # example: http://c6401.ambari.apache.org:8886/solr
NUM_SHARDS=... # use that number that was used for the old collection - important to use at least that many that you have originally before backup
NUM_REP=1 # can be more, but 1 is recommended for that temp collection
MAX_SHARDS_PER_NODE=... # use that number that was used for the old collection
CONFIG_NAME=old_ranger_audits
OLD_DATA_COLLECTION=old_ranger_audits

# kinit only if kerberos is enabled for tha cluster
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)

export JAVA_HOME=/usr/jdk64/1.8.0_112 # or other jdk8 location
export ZK_CONN_STR=... # without znode, e.g.: myhost1:2181,myhost2:2181,myhost3:2181

# note 1: jaas-file option required only if kerberos is enabled for the cluster
# note 2: copy new solrconfig.xml as the old one won't be compatible with solr 7
infra-solr-cloud-cli --transfer-znode -z $ZK_CONN_STR --jaas-file /etc/ambari-infra-solr/conf/infra_solr_jaas.conf --copy-src /infra-solr/configs/ranger_audits/solrconfig.xml --copy-dest /infra-solr/configs/old_ranger_audits/solrconfig.xml

curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=CREATE&name=$OLD_DATA_COLLECTION&numShards=$NUM_SHARDS&replicationFactor=$NUM_REP&maxShardsPerNode=$MAX_SHARDS_PER_NODE&collection.configName=$CONFIG_NAME"
```

Then restore the cores with Solr REST API: ([get core names](#get-core-/-shard-names-with-hosts))

```bash
su infra-solr
SOLR_URL=... # actual solr host url, example: http://c6401.ambari.apache.org:8886/solr
BACKUP_PATH=... # backup location, e.g.: /tmp/ranger-backup

OLD_BACKUP_COLLECTION_CORE=... # choose a core to restore
BACKUP_CORE_NAME=... # choose a core from backup cores - you can find these names as : <backup_location>/snapshot.$BACKUP_CORE_NAME

kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)
curl --negotiate -k -u : "$SOLR_URL/$OLD_BACKUP_COLLECTION_CORE/replication?command=RESTORE&location=$BACKUP_PATH&name=$BACKUP_CORE_NAME"
```

Or use simple `cp` or `hdfs dfs -put` commands to copy the migrated cores to the right places.

#### <a id="vi/4.-restore-old-atlas-collections">VII/2. Restore Old Atlas collections</a>

For Atlas, use `old_` prefix for all 3 collections that you need to create  and use `atlas_configs` config set, then use those for restore the backups;

```bash
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action restore -s ATLAS
```

Or you can do the create collection and restore collections (cores) step by step:

Create a collection for restoring the backup (`old_fulltext_index`, `old_vertex_index`, `old_edge_index`)
```bash
su infra-solr # infra-solr user - if you have a custom one, use that
SOLR_URL=... # example: http://c6401.ambari.apache.org:8886/solr
NUM_SHARDS=... # use that number that was used for the old collection - important to use at least that many that you have originally before backup
NUM_REP=1 # use 1!
MAX_SHARDS_PER_NODE=... # use that number that was used for the old collection
CONFIG_NAME=atlas_configs

# kinit only if kerberos is enabled for tha cluster
kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)

OLD_DATA_COLLECTION=old_fulltext_index
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=CREATE&name=$OLD_DATA_COLLECTION&numShards=$NUM_SHARDS&replicationFactor=$NUM_REP&maxShardsPerNode=$MAX_SHARDS_PER_NODE&collection.configName=$CONFIG_NAME"
OLD_DATA_COLLECTION=old_edge_index
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=CREATE&name=$OLD_DATA_COLLECTION&numShards=$NUM_SHARDS&replicationFactor=$NUM_REP&maxShardsPerNode=$MAX_SHARDS_PER_NODE&collection.configName=$CONFIG_NAME"
OLD_DATA_COLLECTION=old_vertex_index
curl --negotiate -k -u : "$SOLR_URL/admin/collections?action=CREATE&name=$OLD_DATA_COLLECTION&numShards=$NUM_SHARDS&replicationFactor=$NUM_REP&maxShardsPerNode=$MAX_SHARDS_PER_NODE&collection.configName=$CONFIG_NAME"
```

Also you can manually run restore commands: ([get core names](#get-core-/-shard-names-with-hosts))

```bash
su infra-solr
SOLR_URL=... # actual solr host url, example: http://c6401.ambari.apache.org:8886/solr
BACKUP_PATH=... # backup location, e.g.: /tmp/fulltext_index-backup

OLD_BACKUP_COLLECTION_CORE=... # choose a core to restore
BACKUP_CORE_NAME=... # choose a core from backup cores - you can find these names as : <backup_location>/snapshot.$BACKUP_CORE_NAME

kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab $(whoami)/$(hostname -f)
curl --negotiate -k -u : "$SOLR_URL/$OLD_BACKUP_COLLECTION_CORE/replication?command=RESTORE&location=$BACKUP_PATH&name=$BACKUP_CORE_NAME"
```

Or use simple `cp` or `hdfs dfs -put` commands to copy the migrated cores to the right places.

### <a id="vii.-restart-infra-solr-instances">VIII. Restart Infra Solr instances</a>

Next step is to restart Solr instances. That can be done on the Ambari UI, or optionally you can use the migrationHelper script for that as well (rolling restart)
```bash
# --batch-interval -> interval between restart solr tasks
/usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action rolling-restart-solr --batch-interval 60
```

### <a id="viii.-transport-old-data-to-new-collections">IX. Transport old data to new collections</a>

Last step (that can be done any time, as you already have your data in Solr) is to transport all data from the backup collections to the live ones.

#### <a id="viii/1.-transport-old-data-to-ranger-collection">IX/1. Transport old data to Ranger collection</a>

In the end, you end up with 2 collections (ranger_audits and old_ranger_audits), in order to drop the restored one, you will need to transfer your old data to the new collection. To achieve this, you can use [solrDataManager.py](#solr-data-manager-script), which is located next to the `migrationHelper.py` script

```bash
# Init values:
SOLR_URL=... # example: http://c6401.ambari.apache.org:8886/solr
INFRA_SOLR_KEYTAB=... # example: /etc/security/keytabs/ambari-infra-solr.service.keytab
INFRA_SOLR_PRINCIPAL=... # example: infra-solr/$(hostname -f)@EXAMPLE.COM
END_DATE=... # example: 2018-02-18T12:00:00.000Z , date until you export data

OLD_COLLECTION=old_ranger_audits
ACTIVE_COLLECTION=ranger_audits
EXCLUDE_FIELDS=_version_ # comma separated exclude fields, at least _version_ is required

DATE_FIELD=evtTime
# infra-solr-data-manager is a symlink points to /usr/lib/ambari-infra-solr-client/solrDataManager.py
infra-solr-data-manager -m archive -v -c $OLD_COLLECTION -s $SOLR_URL -z none -r 10000 -w 100000 -f $DATE_FIELD -e $END_DATE --solr-output-collection $ACTIVE_COLLECTION -k $INFRA_SOLR_KEYTAB -n $INFRA_SOLR_PRINCIPAL --exclude-fields $EXCLUDE_FIELDS

# Or if you want to run the command in the background (with log and pid file):
nohup infra-solr-data-manager -m archive -v -c $OLD_COLLECTION -s $SOLR_URL -z none -r 10000 -w 100000 -f $DATE_FIELD -e $END_DATE --solr-output-collection $ACTIVE_COLLECTION -k $INFRA_SOLR_KEYTAB -n $INFRA_SOLR_PRINCIPAL --exclude-fields $EXCLUDE_FIELDS > /tmp/solr-data-mgr.log 2>&1>& echo $! > /tmp/solr-data-mgr.pid
```

#### <a id="viii/2.-transport-old-data-to-atlas-collections">IX/2. Transport old data to Atlas collections</a>

In the end, you end up with 6 Atlas collections (vertex_index, old_vertex_index, edge_index, old_edge_index, fulltext_index, old_fulltext_index), in order to drop the restored one, you will need to transfer your old data to the new collection. To achieve this, you can use [solrDataManager.py](#solr-data-manager-script), which is located next to the `migrationHelper.py` script

Example: (with fulltext_index, to the same with edge_index and vertex_index)
```bash
# Init values:
SOLR_URL=... # example: http://c6401.ambari.apache.org:8886/solr
INFRA_SOLR_KEYTAB=... # example: /etc/security/keytabs/ambari-infra-solr.service.keytab
INFRA_SOLR_PRINCIPAL=... # example: infra-solr/$(hostname -f)@EXAMPLE.COM
END_DATE=... # example: 2018-02-18T12:00:00.000Z , date until you export data

OLD_COLLECTION=old_fulltext_index
ACTIVE_COLLECTION=fulltext_index
EXCLUDE_FIELDS=_version_ # comma separated exclude fields, at least _version_ is required

DATE_FIELD=timestamp
# infra-solr-data-manager is a symlink points to /usr/lib/ambari-infra-solr-client/solrDataManager.py
infra-solr-data-manager -m archive -v -c $OLD_COLLECTION -s $SOLR_URL -z none -r 10000 -w 100000 -f $DATE_FIELD -e $END_DATE --solr-output-collection $ACTIVE_COLLECTION -k $INFRA_SOLR_KEYTAB -n $INFRA_SOLR_PRINCIPAL --exclude-fields $EXCLUDE_FIELDS

# Or if you want to run the command in the background (with log and pid file):
nohup infra-solr-data-manager -m archive -v -c $OLD_COLLECTION -s $SOLR_URL -z none -r 10000 -w 100000 -f $DATE_FIELD -e $END_DATE --solr-output-collection $ACTIVE_COLLECTION -k $INFRA_SOLR_KEYTAB -n $INFRA_SOLR_PRINCIPAL --exclude-fields $EXCLUDE_FIELDS > /tmp/solr-data-mgr.log 2>&1>& echo $! > /tmp/solr-data-mgr.pid
```

### <a id="happy-path">Happy path</a>

```bash
CONFIG_INI_LOCATION=ambari_migration.ini
BACKUP_BASE_PATH=/tmp

# if backup is required:
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationConfigGenerator.py --ini-file $CONFIG_INI_LOCATION --host c7401.ambari.apache.org -port 8080 --cluster cl1 --username admin --password admin --backup-base-path=$BACKUP_BASE_PATH --java-home /usr/jdk64/jdk1.8.0_112
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action backup-and-cleanup
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action migrate
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action restore
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action rolling-restart-solr

# or if backup is not required:
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationConfigGenerator.py --ini-file $CONFIG_INI_LOCATION --host c7401.ambari.apache.org -port 8080 --cluster cl1 --username admin --password admin --backup-base-path=$BACKUP_BASE_PATH --java-home /usr/jdk64/jdk1.8.0_112
/usr/bin/python /usr/lib/ambari-infra-solr-client/migrationHelper.py --ini-file $CONFIG_INI_LOCATION --action delete-collections
```

### <a id="appendix">APPENDIX</a>

#### <a id="get-core-/-shard-names-with-hosts">Get core / shard names with hosts</a>

To get which hosts are related for your collections, you can check the Solr UI (using SPNEGO), or checkout get state.json details using a zookeeper-client or Solr zookeeper api to get state.json details of the collection (`/solr/admin/zookeeper?detail=true&path=/collections/<collection_name>/state.json`)

#### <a id="turn-off-infra-solr-authorization">Turn off Infra Solr Authorization</a>

You can turn off Solr authorization plugin with setting `infra-solr-security-json/content` Ambari configuration to `{"authentication": {"class": "org.apache.solr.security.KerberosPlugin"}}` (with that authentication will be still enabled). Then you will need to restart Solr, as that config is uploaded to the `/infra-solr/security.json` znode during startup. Other option is to use zkcli.sh of an Infra Solr to upload the security.json to the right place:
```bash
# Setup env for zkcli.sh
source /etc/ambari-infra-solr/conf/infra-solr-env.sh
# Run that command only if kerberos is enabled.
export SOLR_ZK_CREDS_AND_ACLS="${SOLR_AUTHENTICATION_OPTS}"
ZK_CONN_STRING=... # connection string -> zookeeper server addresses with the znode, e.g.: c7401.ambari.apache.org:2181/infra-solr

/usr/lib/ambari-infra-solr/server/scripts/cloud-scripts/zkcli.sh -zkhost $ZK_CONN_STRING -cmd put /security.json
  '{"authentication": {"class": "org.apache.solr.security.KerberosPlugin"}}'
```

#### <a id="">Solr Migration Helper Script</a>

`/usr/lib/ambari-infra-solr-client/migrationHelper.py --help`

```text
Usage: migrationHelper.py [options]

Options:
-h, --help            show this help message and exit
  -a ACTION, --action=ACTION
                        delete-collections | backup | cleanup-znodes | backup-
                        and-cleanup | migrate | restore | rolling-restart-solr
  -i INI_FILE, --ini-file=INI_FILE
                        Config ini file to parse (required)
  -f, --force           force index upgrade even if it's the right version
  -v, --verbose         use for verbose logging
  -s SERVICE_FILTER, --service-filter=SERVICE_FILTER
                        run commands only selected services (comma separated:
                        LOGSEARCH,ATLAS,RANGER)
  -c COLLECTION, --collection=COLLECTION
                        selected collection to run an operation
  --async               async Ambari operations (backup | restore | migrate)
  --index-location=INDEX_LOCATION
                        location of the index backups. add ranger/atlas prefix
                        after the path. required only if no backup path in the
                        ini file
  --atlas-index-location=ATLAS_INDEX_LOCATION
                        location of the index backups (for atlas). required
                        only if no backup path in the ini file
  --ranger-index-location=RANGER_INDEX_LOCATION
                        location of the index backups (for ranger). required
                        only if no backup path in the ini file
  --version=INDEX_VERSION
                        lucene index version for migration (6.6.2 or 7.3.1)
  --request-tries=REQUEST_TRIES
                        number of tries for BACKUP/RESTORE status api calls in
                        the request
  --request-time-interval=REQUEST_TIME_INTERVAL
                        time interval between BACKUP/RESTORE status api calls
                        in the request
  --request-async       skip BACKUP/RESTORE status api calls from the command
  --include-solr-hosts=INCLUDE_SOLR_HOSTS
                        comma separated list of included solr hosts
  --exclude-solr-hosts=EXCLUDE_SOLR_HOSTS
                        comma separated list of excluded solr hosts
  --disable-solr-host-check
                        Disable to check solr hosts are good for the
                        collection backups
  --core-filter=CORE_FILTER
                        core filter for replica folders
  --skip-cores=SKIP_CORES
                        specific cores to skip (comma separated)
  --skip-generate-restore-host-cores
                        Skip the generation of restore_host_cores.json, just
                        read the file itself, can be useful if command failed
                        at some point.
  --hdfs-base-path=HDFS_BASE_PATH
                        hdfs base path where the collections are located
                        (e.g.: /user/infrasolr). Use if both atlas and ranger
                        collections are on hdfs.
  --ranger-hdfs-base-path=RANGER_HDFS_BASE_PATH
                        hdfs base path where the ranger collection is located
                        (e.g.: /user/infra-solr). Use if only ranger
                        collection is on hdfs.
  --atlas-hdfs-base-path=ATLAS_HDFS_BASE_PATH
                        hdfs base path where the atlas collections are located
                        (e.g.: /user/infra-solr). Use if only atlas
                        collections are on hdfs.
  --keep-backup         If it is turned on, Snapshot Solr data will not be
                        deleted from the filesystem during restore.
  --batch-interval=BATCH_INTERVAL
                        batch time interval (seconds) between requests (for
                        restarting INFRA SOLR, default: 60)
  --batch-fault-tolerance=BATCH_FAULT_TOLERANCE
                        fault tolerance of tasks for batch request (for
                        restarting INFRA SOLR, default: 0)
  --shared-drive        Use if the backup location is shared between hosts.
                        (override config from config ini file)
```

#### <a id="migration-config-generator">Solr Migration Config Generator Script</a>

```text
Usage: migrationConfigGenerator.py [options]

Options:
  -h, --help            show this help message and exit
  -H HOST, --host=HOST  hostname for ambari server
  -P PORT, --port=PORT  port number for ambari server
  -c CLUSTER, --cluster=CLUSTER
                        name cluster
  -f, --force-ranger    force to get Ranger details - can be useful if Ranger
                        is configured to use external Solr (but points to
                        internal Sols)
  -s, --ssl             use if ambari server using https
  -v, --verbose         use for verbose logging
  -u USERNAME, --username=USERNAME
                        username for accessing ambari server
  -p PASSWORD, --password=PASSWORD
                        password for accessing ambari server
  -j JAVA_HOME, --java-home=JAVA_HOME
                        local java_home location
  -i INI_FILE, --ini-file=INI_FILE
                        Filename of the generated ini file for migration
                        (default: ambari_solr_migration.ini)
  --backup-base-path=BACKUP_BASE_PATH
                        base path for backup, e.g.: /tmp/backup, then
                        /tmp/backup/ranger/ and /tmp/backup/atlas/ folders
                        will be generated
  --backup-ranger-base-path=BACKUP_RANGER_BASE_PATH
                        base path for ranger backup (override backup-base-path 
                        for ranger), e.g.: /tmp/backup/ranger
  --backup-atlas-base-path=BACKUP_ATLAS_BASE_PATH
                        base path for atlas backup (override backup-base-path 
                        for ranger), e.g.: /tmp/backup/atlas
  --hdfs-base-path=HDFS_BASE_PATH
                        hdfs base path where the collections are located
                        (e.g.: /user/infrasolr). Use if both atlas and ranger
                        collections are on hdfs.
  --ranger-hdfs-base-path=RANGER_HDFS_BASE_PATH
                        hdfs base path where the ranger collection is located
                        (e.g.: /user/infra-solr). Use if only ranger
                        collection is on hdfs.
  --atlas-hdfs-base-path=ATLAS_HDFS_BASE_PATH
                        hdfs base path where the atlas collections are located
                        (e.g.: /user/infra-solr). Use if only atlas
                        collections are on hdfs.
  --skip-atlas          skip to gather Atlas service details
  --skip-ranger         skip to gather Ranger service details
  --retry=RETRY         number of retries during accessing random solr urls
  --delay=DELAY         delay (seconds) between retries during accessing
                        random solr urls
  --shared-drive        Use if the backup location is shared between hosts.
```

#### <a id="">Solr Data Manager Script</a>

`/usr/lib/ambari-infra-solr-client/solrDataManager.py --help`

```text
Usage: solrDataManager.py [options]

Options:
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -m MODE, --mode=MODE  archive | delete | save
  -s SOLR_URL, --solr-url=SOLR_URL
                        the url of the solr server including the port
  -c COLLECTION, --collection=COLLECTION
                        the name of the solr collection
  -f FILTER_FIELD, --filter-field=FILTER_FIELD
                        the name of the field to filter on
  -r READ_BLOCK_SIZE, --read-block-size=READ_BLOCK_SIZE
                        block size to use for reading from solr
  -w WRITE_BLOCK_SIZE, --write-block-size=WRITE_BLOCK_SIZE
                        number of records in the output files
  -i ID_FIELD, --id-field=ID_FIELD
                        the name of the id field
  -o DATE_FORMAT, --date-format=DATE_FORMAT
                        the date format to use for --days
  -q ADDITIONAL_FILTER, --additional-filter=ADDITIONAL_FILTER
                        additional solr filter
  -j NAME, --name=NAME  name included in result files
  -g, --ignore-unfinished-uploading
  --json-file           create a json file instead of line delimited json
  -z COMPRESSION, --compression=COMPRESSION
                        none | tar.gz | tar.bz2 | zip | gz
  -k SOLR_KEYTAB, --solr-keytab=SOLR_KEYTAB
                        the keytab for a kerberized solr
  -n SOLR_PRINCIPAL, --solr-principal=SOLR_PRINCIPAL
                        the principal for a kerberized solr
  -a HDFS_KEYTAB, --hdfs-keytab=HDFS_KEYTAB
                        the keytab for a kerberized hdfs
  -l HDFS_PRINCIPAL, --hdfs-principal=HDFS_PRINCIPAL
                        the principal for a kerberized hdfs
  -u HDFS_USER, --hdfs-user=HDFS_USER
                        the user for accessing hdfs
  -p HDFS_PATH, --hdfs-path=HDFS_PATH
                        the hdfs path to upload to
  -t KEY_FILE_PATH, --key-file-path=KEY_FILE_PATH
                        the file that contains S3 <accessKey>,<secretKey>
  -b BUCKET, --bucket=BUCKET
                        the bucket name for S3 upload
  -y KEY_PREFIX, --key-prefix=KEY_PREFIX
                        the key prefix for S3 upload
  -x LOCAL_PATH, --local-path=LOCAL_PATH
                        the local path to save the files to
  -v, --verbose
  --solr-output-collection=SOLR_OUTPUT_COLLECTION
                        target output solr collection for archive
  --exclude-fields=EXCLUDE_FIELDS
                        Comma separated list of excluded fields from json
                        response

  specifying the end of the range:
    -e END, --end=END     end of the range
    -d DAYS, --days=DAYS  number of days to keep
```
