---
machine_translated: true
machine_translated_rev: 72537a2d527c63c07aa5d2361a8829f3895cf2bd
toc_priority: 59
toc_title: clickhouse-copieur
---

# clickhouse-copieur {#clickhouse-copier}

Copie les données des tables d'un cluster vers des tables d'un autre cluster (ou du même cluster).

Vous pouvez exécuter plusieurs `clickhouse-copier` instances sur différents serveurs pour effectuer le même travail. ZooKeeper est utilisé pour synchroniser les processus.

Après le démarrage de, `clickhouse-copier`:

-   Se connecte à ZooKeeper et reçoit:

    -   La copie de tâches.
    -   L'état de la copie d'emplois.

-   Il effectue les travaux.

    Chaque processus en cours choisit le “closest” eclat du cluster source et copie les données dans le cluster de destination, la refragmentation les données si nécessaire.

`clickhouse-copier` suit les changements dans ZooKeeper et les applique à la volée.

Pour réduire le trafic réseau, nous vous recommandons de `clickhouse-copier` sur le même serveur où se trouvent les données source.

## Course Clickhouse-copieur {#running-clickhouse-copier}

L'utilitaire doit être exécuté manuellement:

``` bash
$ clickhouse-copier copier --daemon --config zookeeper.xml --task-path /task/path --base-dir /path/to/dir
```

Paramètre:

-   `daemon` — Starts `clickhouse-copier` en mode démon.
-   `config` — The path to the `zookeeper.xml` fichier avec les paramètres pour la connexion à la Gardienne.
-   `task-path` — The path to the ZooKeeper node. This node is used for syncing `clickhouse-copier` processus et stockage des tâches. Les tâches sont stockées dans `$task-path/description`.
-   `task-file` — Optional path to file with task configuration for initial upload to ZooKeeper.
-   `task-upload-force` — Force upload `task-file` même si le nœud existe déjà.
-   `base-dir` — The path to logs and auxiliary files. When it starts, `clickhouse-copier` crée `clickhouse-copier_YYYYMMHHSS_<PID>` les sous-répertoires `$base-dir`. Si ce paramètre est omis, les répertoires sont créés dans le répertoire où `clickhouse-copier` a été lancé.

## Format de Zookeeper.XML {#format-of-zookeeper-xml}

``` xml
<yandex>
    <logger>
        <level>trace</level>
        <size>100M</size>
        <count>3</count>
    </logger>

    <zookeeper>
        <node index="1">
            <host>127.0.0.1</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>
```

## Configuration des tâches de copie {#configuration-of-copying-tasks}

``` xml
<yandex>
    <!-- Configuration of clusters as in an ordinary server config -->
    <remote_servers>
        <source_cluster>
            <shard>
                <internal_replication>false</internal_replication>
                    <replica>
                        <host>127.0.0.1</host>
                        <port>9000</port>
                    </replica>
            </shard>
            ...
        </source_cluster>

        <destination_cluster>
        ...
        </destination_cluster>
    </remote_servers>

    <!-- How many simultaneously active workers are possible. If you run more workers superfluous workers will sleep. -->
    <max_workers>2</max_workers>

    <!-- Setting used to fetch (pull) data from source cluster tables -->
    <settings_pull>
        <readonly>1</readonly>
    </settings_pull>

    <!-- Setting used to insert (push) data to destination cluster tables -->
    <settings_push>
        <readonly>0</readonly>
    </settings_push>

    <!-- Common setting for fetch (pull) and insert (push) operations. Also, copier process context uses it.
         They are overlaid by <settings_pull/> and <settings_push/> respectively. -->
    <settings>
        <connect_timeout>3</connect_timeout>
        <!-- Sync insert is set forcibly, leave it here just in case. -->
        <insert_distributed_sync>1</insert_distributed_sync>
    </settings>

    <!-- Copying tasks description.
         You could specify several table task in the same task description (in the same ZooKeeper node), they will be performed
         sequentially.
    -->
    <tables>
        <!-- A table task, copies one table. -->
        <table_hits>
            <!-- Source cluster name (from <remote_servers/> section) and tables in it that should be copied -->
            <cluster_pull>source_cluster</cluster_pull>
            <database_pull>test</database_pull>
            <table_pull>hits</table_pull>

            <!-- Destination cluster name and tables in which the data should be inserted -->
            <cluster_push>destination_cluster</cluster_push>
            <database_push>test</database_push>
            <table_push>hits2</table_push>

            <!-- Engine of destination tables.
                 If destination tables have not be created, workers create them using columns definition from source tables and engine
                 definition from here.

                 NOTE: If the first worker starts insert data and detects that destination partition is not empty then the partition will
                 be dropped and refilled, take it into account if you already have some data in destination tables. You could directly
                 specify partitions that should be copied in <enabled_partitions/>, they should be in quoted format like partition column of
                 system.parts table.
            -->
            <engine>
            ENGINE=ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/hits2', '{replica}')
            PARTITION BY toMonday(date)
            ORDER BY (CounterID, EventDate)
            </engine>

            <!-- Sharding key used to insert data to destination cluster -->
            <sharding_key>jumpConsistentHash(intHash64(UserID), 2)</sharding_key>

            <!-- Optional expression that filter data while pull them from source servers -->
            <where_condition>CounterID != 0</where_condition>

            <!-- This section specifies partitions that should be copied, other partition will be ignored.
                 Partition names should have the same format as
                 partition column of system.parts table (i.e. a quoted text).
                 Since partition key of source and destination cluster could be different,
                 these partition names specify destination partitions.

                 NOTE: In spite of this section is optional (if it is not specified, all partitions will be copied),
                 it is strictly recommended to specify them explicitly.
                 If you already have some ready partitions on destination cluster they
                 will be removed at the start of the copying since they will be interpeted
                 as unfinished data from the previous copying!!!
            -->
            <enabled_partitions>
                <partition>'2018-02-26'</partition>
                <partition>'2018-03-05'</partition>
                ...
            </enabled_partitions>
        </table_hits>

        <!-- Next table to copy. It is not copied until previous table is copying. -->
        </table_visits>
        ...
        </table_visits>
        ...
    </tables>
</yandex>
```

`clickhouse-copier` suit les changements dans `/task/path/description` et les applique à la volée. Par exemple, si vous modifiez la valeur de `max_workers`, le nombre de processus exécutant des tâches changera également.

[Article Original](https://clickhouse.tech/docs/en/operations/utils/clickhouse-copier/) <!--hide-->
