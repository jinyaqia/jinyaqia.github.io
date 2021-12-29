# clickhouse集群2副本安装

{:toc}

* TOC
## 使用4台机器

```
节点1  10.64.148.133
节点2  10.64.148.134
节点3  10.64.148.135
节点4  10.64.138.24
```

## 配置路径

```
root@ip-10-64-138-24:/data/clickhouse# tree
.
├── bin
│   ├── clickhouse
│   ├── clickhouse-client -> /data/clickhouse/bin/clickhouse
│   └── clickhouse-server -> /data/clickhouse/bin/clickhouse
├── conf
│   ├── config.d
│   │   ├── config.base.xml
│   │   ├── config.disks.xml
│   │   ├── config.macros.xml
│   │   └── config.remote_servers.xml
│   ├── config.xml
│   ├── env.ini
│   ├── users.d
│   │   └── users.huya.xml
│   └── users.xml
├── format_schemas
├── log
│   ├── clickhouse-server.err.log
│   ├── clickhouse-server.log
│   ├── stderr.log
│   └── stdout.log
├── start_ch.sh
├── tmp
└── user_files
```

- `/data/clickhouse`为安装目录

- `bin/`下为自行编译的clickhouse二进制文件

- `conf/`下为ck的配置, `config.xml`为启动进程时默认加载的，
  - `config.d/*.xml`为具体的子配置，具体包括
    - `config.base.xml`: 基础配置
    - `config.disks.xml`: 磁盘和存储策略配置
    - `config.macros.xml`:宏定义配置
    - `config.remote_servers.xml`: 集群配置
  - `env.ini`： 环境配置
- `users.d`: 存储用户配置，访问权限之类
- `start_ch.sh`:启动脚本

对应每个文件如下：

### 默认启动配置 /data/clickhouse/conf/config.xml

```
<?xml version="1.0"?>
<!--
  NOTE: User and query level settings are set up in "users.xml" file.
-->
<yandex>
    <logger>
        <!-- Possible levels: https://github.com/pocoproject/poco/blob/poco-1.9.4-release/Foundation/include/Poco/Logger.h#L105 -->
        <level>trace</level>
        <log>/data/clickhouse/log/clickhouse-server.log</log>
        <errorlog>/data/clickhouse/log/clickhouse-server.err.log</errorlog>
        <size>1000M</size>
        <count>10</count>
        <!-- <console>1</console> --> <!-- Default behavior is autodetection (log to console if not daemon mode and is tty) -->
    </logger>
    <!--display_name>production</display_name--> <!-- It is the name that will be shown in the client -->
    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>
    <mysql_port>9004</mysql_port>
    <!-- For HTTPS and SSL over native protocol. -->
    <!--
    <https_port>8443</https_port>
    <tcp_port_secure>9440</tcp_port_secure>
    -->

    <!-- Used with https_port and tcp_port_secure. Full ssl options list: https://github.com/ClickHouse-Extras/poco/blob/master/NetSSL_OpenSSL/include/Poco/Net/SSLManager.h#L71 -->
    <openSSL>
        <server> <!-- Used for https server AND secure tcp port -->
            <!-- openssl req -subj "/CN=localhost" -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/clickhouse-server/server.key -out /etc/clickhouse-server/server.crt -->
            <certificateFile>/etc/clickhouse-server/server.crt</certificateFile>
            <privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>
            <!-- openssl dhparam -out /etc/clickhouse-server/dhparam.pem 4096 -->
            <dhParamsFile>/etc/clickhouse-server/dhparam.pem</dhParamsFile>
            <verificationMode>none</verificationMode>
            <loadDefaultCAFile>true</loadDefaultCAFile>
            <cacheSessions>true</cacheSessions>
            <disableProtocols>sslv2,sslv3</disableProtocols>
            <preferServerCiphers>true</preferServerCiphers>
        </server>

        <client> <!-- Used for connecting to https dictionary source -->
            <loadDefaultCAFile>true</loadDefaultCAFile>
            <cacheSessions>true</cacheSessions>
            <disableProtocols>sslv2,sslv3</disableProtocols>
            <preferServerCiphers>true</preferServerCiphers>
            <!-- Use for self-signed: <verificationMode>none</verificationMode> -->
            <invalidCertificateHandler>
                <!-- Use for self-signed: <name>AcceptCertificateHandler</name> -->
                <name>RejectCertificateHandler</name>
            </invalidCertificateHandler>
        </client>
    </openSSL>

    <!-- Default root page on http[s] server. For example load UI from https://tabix.io/ when opening http://localhost:8123 -->
    <!--
    <http_server_default_response><![CDATA[<html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>]]></http_server_default_response>
    -->

    <!-- Port for communication between replicas. Used for data exchange. -->
    <interserver_http_port>9009</interserver_http_port>

    <!-- Hostname that is used by other replicas to request this server.
         If not specified, than it is determined analoguous to 'hostname -f' command.
         This setting could be used to switch replication to another network interface.
      -->
    <!--
    <interserver_http_host>example.yandex.ru</interserver_http_host>
    -->

    <!-- Listen specified host. use :: (wildcard IPv6 address), if you want to accept connections both with IPv4 and IPv6 from everywhere. -->
    <!-- <listen_host>::</listen_host> -->
    <!-- Same for hosts with disabled ipv6: -->
    <!-- <listen_host>0.0.0.0</listen_host> -->

    <!-- Default values - try listen localhost on ipv4 and ipv6: -->

    <listen_host>0.0.0.0</listen_host>
    <!-- <listen_host>127.0.0.1</listen_host> -->

    <!-- Don't exit if ipv6 or ipv4 unavailable, but listen_host with this protocol specified -->
    <!-- <listen_try>0</listen_try> -->

    <!-- Allow listen on same address:port -->
    <!-- <listen_reuse_port>0</listen_reuse_port> -->

    <!-- <listen_backlog>64</listen_backlog> -->

    <max_connections>4096</max_connections>
    <keep_alive_timeout>3</keep_alive_timeout>

    <!-- Maximum number of concurrent queries. -->
    <max_concurrent_queries>500</max_concurrent_queries>

    <!-- Set limit on number of open files (default: maximum). This setting makes sense on Mac OS X because getrlimit() fails to retrieve
         correct maximum value. -->
    <!-- <max_open_files>262144</max_open_files> -->

    <!-- Size of cache of uncompressed blocks of data, used in tables of MergeTree family.
         In bytes. Cache is single for server. Memory is allocated only on demand.
         Cache is used when 'use_uncompressed_cache' user setting turned on (off by default).
         Uncompressed cache is advantageous only for very short queries and in rare cases.
      -->
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>

    <!-- Approximate size of mark cache, used in tables of MergeTree family.
         In bytes. Cache is single for server. Memory is allocated only on demand.
         You should not lower this value.
      -->
    <mark_cache_size>5368709120</mark_cache_size>


    <!-- Path to data directory, with trailing slash. -->
    <path>/data/clickhouse/</path>

    <!-- Path to temporary data for processing hard queries. -->
    <tmp_path>/data/clickhouse/tmp/</tmp_path>

    <!-- Policy from the <storage_configuration> for the temporary files.
         If not set <tmp_path> is used, otherwise <tmp_path> is ignored.

         Notes:
         - move_factor              is ignored
         - keep_free_space_bytes    is ignored
         - max_data_part_size_bytes is ignored
         - you must have exactly one volume in that policy
    -->
    <!-- <tmp_policy>tmp</tmp_policy> -->

    <!-- Directory with user provided files that are accessible by 'file' table function. -->
    <user_files_path>/data/clickhouse/user_files/</user_files_path>

    <!-- Path to configuration file with users, access rights, profiles of settings, quotas. -->
    <users_config>users.xml</users_config>

    <!-- Default profile of settings. -->
    <default_profile>default</default_profile>

    <!-- System profile of settings. This settings are used by internal processes (Buffer storage, Distibuted DDL worker and so on). -->
    <!-- <system_profile>default</system_profile> -->

    <!-- Default database. -->
    <default_database>default</default_database>

    <!-- Server time zone could be set here.

         Time zone is used when converting between String and DateTime types,
          when printing DateTime in text formats and parsing DateTime from text,
          it is used in date and time related functions, if specific time zone was not passed as an argument.

         Time zone is specified as identifier from IANA time zone database, like UTC or Africa/Abidjan.
         If not specified, system time zone at server startup is used.

         Please note, that server could display time zone alias instead of specified name.
         Example: W-SU is an alias for Europe/Moscow and Zulu is an alias for UTC.
    -->
    <timezone>Asia/Shanghai</timezone>

    <!-- You can specify umask here (see "man umask"). Server will apply it on startup.
         Number is always parsed as octal. Default umask is 027 (other users cannot read logs, data files, etc; group can only read).
    -->
    <!-- <umask>022</umask> -->

    <!-- Perform mlockall after startup to lower first queries latency
          and to prevent clickhouse executable from being paged out under high IO load.
         Enabling this option is recommended but will lead to increased startup time for up to a few seconds.
    -->
    <mlock_executable>false</mlock_executable>
    <!-- Configuration of clusters that could be used in Distributed tables.
         https://clickhouse.tech/docs/en/operations/table_engines/distributed/
      -->
    <remote_servers incl="clickhouse_remote_servers" >
        <!-- Test only shard config for testing distributed storage -->
    </remote_servers>

    <!-- The list of hosts allowed to use in URL-related storage engines and table functions.
        If this section is not present in configuration, all hosts are allowed.
    -->
    <remote_url_allow_hosts>
        <!-- Host should be specified exactly as in URL. The name is checked before DNS resolution.
            Example: "yandex.ru", "yandex.ru." and "www.yandex.ru" are different hosts.
                    If port is explicitly specified in URL, the host:port is checked as a whole.
                    If host specified here without port, any port with this host allowed.
                    "yandex.ru" -> "yandex.ru:443", "yandex.ru:80" etc. is allowed, but "yandex.ru:80" -> only "yandex.ru:80" is allowed.
            If the host is specified as IP address, it is checked as specified in URL. Example: "[2a02:6b8:a::a]".
            If there are redirects and support for redirects is enabled, every redirect (the Location field) is checked.
        -->

        <!-- Regular expression can be specified. RE2 engine is used for regexps.
            Regexps are not aligned: don't forget to add ^ and $. Also don't forget to escape dot (.) metacharacter
            (forgetting to do so is a common source of error).
        -->
    </remote_url_allow_hosts>

    <!-- If element has 'incl' attribute, then for it's value will be used corresponding substitution from another file.
         By default, path to file with substitutions is /etc/metrika.xml. It could be changed in config in 'include_from' element.
         Values for substitutions are specified in /yandex/name_of_substitution elements in that file.
      -->

    <!-- ZooKeeper is used to store metadata about replicas, when using Replicated tables.
         Optional. If you don't use replicated tables, you could omit that.

         See https://clickhouse.yandex/docs/en/table_engines/replication/
      -->

    <zookeeper incl="zookeeper-servers" optional="true"/>

    <!-- Substitutions for parameters of replicated tables.
          Optional. If you don't use replicated tables, you could omit that.

         See https://clickhouse.yandex/docs/en/table_engines/replication/#creating-replicated-tables
      -->
    <macros incl="macros" optional="true" />


    <!-- Reloading interval for embedded dictionaries, in seconds. Default: 3600. -->
    <builtin_dictionaries_reload_interval>3600</builtin_dictionaries_reload_interval>


    <!-- Maximum session timeout, in seconds. Default: 3600. -->
    <max_session_timeout>3600</max_session_timeout>

    <!-- Default session timeout, in seconds. Default: 60. -->
    <default_session_timeout>60</default_session_timeout>

    <!-- Sending data to Graphite for monitoring. Several sections can be defined. -->
    <!--
        interval - send every X second
        root_path - prefix for keys
        hostname_in_path - append hostname to root_path (default = true)
        metrics - send data from table system.metrics
        events - send data from table system.events
        asynchronous_metrics - send data from table system.asynchronous_metrics
    -->
    <!--
    <graphite>
        <host>localhost</host>
        <port>42000</port>
        <timeout>0.1</timeout>
        <interval>60</interval>
        <root_path>one_min</root_path>
        <hostname_in_path>true</hostname_in_path>

        <metrics>true</metrics>
        <events>true</events>
        <events_cumulative>false</events_cumulative>
        <asynchronous_metrics>true</asynchronous_metrics>
    </graphite>
    <graphite>
        <host>localhost</host>
        <port>42000</port>
        <timeout>0.1</timeout>
        <interval>1</interval>
        <root_path>one_sec</root_path>

        <metrics>true</metrics>
        <events>true</events>
        <events_cumulative>false</events_cumulative>
        <asynchronous_metrics>false</asynchronous_metrics>
    </graphite>
    -->

    <!-- Serve endpoint fot Prometheus monitoring. -->
    <!--
        endpoint - mertics path (relative to root, statring with "/")
        port - port to setup server. If not defined or 0 than http_port used
        metrics - send data from table system.metrics
        events - send data from table system.events
        asynchronous_metrics - send data from table system.asynchronous_metrics
    -->
    <!--
    <prometheus>
        <endpoint>/metrics</endpoint>
        <port>9363</port>

        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
    </prometheus>
    -->

    <!-- Query log. Used only for queries with setting log_queries = 1. -->
    <query_log>
        <!-- What table to insert data. If table is not exist, it will be created.
             When query log structure is changed after system update,
              then old table will be renamed and new table will be created automatically.
        -->
        <database>system</database>
        <table>query_log</table>
        <!--
            PARTITION BY expr https://clickhouse.yandex/docs/en/table_engines/custom_partitioning_key/
            Example:
                event_date
                toMonday(event_date)
                toYYYYMM(event_date)
                toStartOfHour(event_time)
        -->
        <partition_by>toYYYYMMDD(event_date)</partition_by>

        <!-- Instead of partition_by, you can provide full engine expression (starting with ENGINE = ) with parameters,
             Example: <engine>ENGINE = MergeTree PARTITION BY toYYYYMM(event_date) ORDER BY (event_date, event_time) SETTINGS index_granularity = 1024</engine>
          -->

        <!-- Interval of flushing data. -->
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_log>

    <!-- Trace log. Stores stack traces collected by query profilers.
         See query_profiler_real_time_period_ns and query_profiler_cpu_time_period_ns settings. -->
    <trace_log>
        <database>system</database>
        <table>trace_log</table>

        <partition_by>toYYYYMMDD(event_date)</partition_by>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </trace_log>

    <!-- Query thread log. Has information about all threads participated in query execution.
         Used only for queries with setting log_query_threads = 1. -->
    <query_thread_log>
        <database>system</database>
        <table>query_thread_log</table>
        <partition_by>toYYYYMM(event_date)</partition_by>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_thread_log>

    <!-- Uncomment if use part log.
         Part log contains information about all actions with parts in MergeTree tables (creation, deletion, merges, downloads).
         -->
    <part_log>
        <database>system</database>
        <table>part_log</table>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </part_log>


    <!-- Uncomment to write text log into table.
         Text log contains all information from usual server log but stores it in structured and efficient way.
         The level of the messages that goes to the table can be limited (<level>), if not specified all messages will go to the table.
           -->
    <text_log>
        <database>system</database>
        <table>text_log</table>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
        <level>trace</level>
    </text_log>


    <!-- Metric log contains rows with current values of ProfileEvents, CurrentMetrics collected with "collect_interval_milliseconds" interval. -->
    <metric_log>
        <database>system</database>
        <table>metric_log</table>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
        <collect_interval_milliseconds>1000</collect_interval_milliseconds>
    </metric_log>

    <!-- Parameters for embedded dictionaries, used in Yandex.Metrica.
         See https://clickhouse.yandex/docs/en/dicts/internal_dicts/
    -->

    <!-- Path to file with region hierarchy. -->
    <!-- <path_to_regions_hierarchy_file>/opt/geo/regions_hierarchy.txt</path_to_regions_hierarchy_file> -->

    <!-- Path to directory with files containing names of regions -->
    <!-- <path_to_regions_names_files>/opt/geo/</path_to_regions_names_files> -->


    <!-- Configuration of external dictionaries. See:
         https://clickhouse.yandex/docs/en/dicts/external_dicts/
    -->
    <dictionaries_config>*_dictionary.xml</dictionaries_config>

    <!-- Uncomment if you want data to be compressed 30-100% better.
         Don't do that if you just started using ClickHouse.
      -->
    <compression incl="clickhouse_compression">
    <!--
        <!- - Set of variants. Checked in order. Last matching case wins. If nothing matches, lz4 will be used. - ->
        <case>

            <!- - Conditions. All must be satisfied. Some conditions may be omitted. - ->
            <min_part_size>10000000000</min_part_size>        <!- - Min part size in bytes. - ->
            <min_part_size_ratio>0.01</min_part_size_ratio>   <!- - Min size of part relative to whole table size. - ->

            <!- - What compression method to use. - ->
            <method>zstd</method>
        </case>
    -->
    </compression>

    <!-- Allow to execute distributed DDL queries (CREATE, DROP, ALTER, RENAME) on cluster.
         Works only if ZooKeeper is enabled. Comment it if such functionality isn't required. -->
    <distributed_ddl>
        <!-- Path in ZooKeeper to queue with DDL queries -->
        <path>/clickhouse/task_queue/ddl</path>

        <!-- Settings from this profile will be used to execute DDL queries -->
        <!-- <profile>default</profile> -->
    </distributed_ddl>

    <!-- Settings to fine tune MergeTree tables. See documentation in source code, in MergeTreeSettings.h -->
    <!--
    <merge_tree>
        <max_suspicious_broken_parts>5</max_suspicious_broken_parts>
    </merge_tree>
    -->

    <!-- Protection from accidental DROP.
         If size of a MergeTree table is greater than max_table_size_to_drop (in bytes) than table could not be dropped with any DROP query.
         If you want do delete one table and don't want to change clickhouse-server config, you could create special file <clickhouse-path>/flags/force_drop_table and make DROP once.
         By default max_table_size_to_drop is 50GB; max_table_size_to_drop=0 allows to DROP any tables.
         The same for max_partition_size_to_drop.
         Uncomment to disable protection.
    -->
    <max_table_size_to_drop>0</max_table_size_to_drop>
    <max_partition_size_to_drop>0</max_partition_size_to_drop>

    <!-- Example of parameters for GraphiteMergeTree table engine -->
    <graphite_rollup_example>
        <pattern>
            <regexp>click_cost</regexp>
            <function>any</function>
            <retention>
                <age>0</age>
                <precision>3600</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>60</precision>
            </retention>
        </pattern>
        <default>
            <function>max</function>
            <retention>
                <age>0</age>
                <precision>60</precision>
            </retention>
            <retention>
                <age>3600</age>
                <precision>300</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>3600</precision>
            </retention>
        </default>
    </graphite_rollup_example>

    <!-- Directory in <clickhouse-path> containing schema files for various input formats.
         The directory will be created if it doesn't exist.
      -->
    <format_schema_path>/data/clickhouse/format_schemas/</format_schema_path>

   

    <!-- Uncomment to use query masking rules.
        name - name for the rule (optional)
        regexp - RE2 compatible regular expression (mandatory)
        replace - substitution string for sensitive data (optional, by default - six asterisks)
    <query_masking_rules>
        <rule>
            <name>hide SSN</name>
            <regexp>\b\d{3}-\d{2}-\d{4}\b</regexp>
            <replace>000-00-0000</replace>
        </rule>
    </query_masking_rules>
    -->

    <!-- Uncomment to disable ClickHouse internal DNS caching. -->
    <!-- <disable_internal_dns_cache>1</disable_internal_dns_cache> -->
</yandex>

```

### 基础配置 /data/clickhouse/conf/config.d/config.base.xml

会覆盖`/data/clickhouse/conf/config.xml`, 每个节点都一样

```
<?xml version="1.0" encoding="UTF-8"?>
<yandex>

    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>
    <listen_host>10.64.138.24</listen_host>
    <interserver_http_port>9009</interserver_http_port>
    <interserver_http_host>10.64.138.24</interserver_http_host>
    <path>/data1/clickhouse</path>
    <tmp_path>/data1/clickhouse/tmp/</tmp_path>
    <max_concurrent_queries>500</max_concurrent_queries>
    <max_table_size_to_drop>0</max_table_size_to_drop>
    <max_partition_size_to_drop>0</max_partition_size_to_drop>
    <zookeeper replace="replace">
        <node index="0">
            <host>10.64.148.133</host>
            <port>2181</port>
        </node>
        <node index="1">
            <host>10.64.148.134</host>
            <port>2181</port>
        </node>
        <node index="1">
            <host>10.64.148.135</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>
```

### 集群分片配置 /data/clickhouse/conf/config.d/config.disks.xml

每个节点都一样

```
<?xml version="1.0" encoding="UTF-8"?>
<yandex>
 <storage_configuration>
        <disks>
            <fast>
                <path>/dev/shm/clickhouse/</path>
                <!-- 10G -->
             <keep_free_space_bytes>10737418240</keep_free_space_bytes>
            </fast>
            <default>
             <keep_free_space_bytes>572159652</keep_free_space_bytes>
            </default>
            <data3>
                <path>/data3/clickhouse/</path>
                <keep_free_space_bytes>572159652</keep_free_space_bytes>
            </data3>
            <data4>
                <path>/data4/clickhouse/</path>
                <keep_free_space_bytes>572159652</keep_free_space_bytes>
            </data4>
            <data5>
                <path>/data5/clickhouse/</path>
                <keep_free_space_bytes>572159652</keep_free_space_bytes>
            </data5>
           
            <data7>
                <path>/data7/clickhouse/</path>
                <keep_free_space_bytes>572159652</keep_free_space_bytes>
            </data7>
            <data8>
                <path>/data8/clickhouse/</path>
                <keep_free_space_bytes>572159652</keep_free_space_bytes>
            </data8>
         
        </disks>
        <policies>
            <hdd_in_order>
                <volumes>
                    <hot>
                       	<disk>default</disk>
                        <disk>data3</disk>
						<max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
                    </hot>
                     <cold>
                        <disk>data4</disk>
                        <disk>data5</disk>
						<max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
                    </cold>
                    <archive>
                        <disk>data7</disk>
                        <disk>data8</disk>
                    </archive>
                </volumes>
                <move_factor>0.2</move_factor>
            </hdd_in_order>
            <memory_cache>
                <volumes>
                    <memory>
                        <disk>fast</disk>
                    </memory>
                </volumes>
            </memory_cache>
        </policies>
    </storage_configuration>
</yandex>
```

###   宏定义 /data/clickhouse/conf/config.d/config.macros.xml

用于执行建表语句时，可以替换成具体的值，不同机器节点不一样:

由于ck的同个节点只能存一个分片的一个副本，所以4台机器可以实现2分片2副本

对于节点1：10.64.148.133   存储分片01 副本 01， shard=01  replica=c1-01-01(表示集群c1, 01分片, 01副本)

```
<?xml version="1.0" encoding="UTF-8"?>
<yandex>
    <macros>
        <shard>01</shard>
        <replica>c1-01-01</replica>
    </macros>
</yandex>
```

对于节点2：10.64.148.134   存储分片01 副本 02， shard=01  replica=c1-01-02(表示集群c1, 01分片, 02副本)

```
<?xml version="1.0" encoding="UTF-8"?>
<yandex>
    <macros>
        <shard>01</shard>
        <replica>c1-01-02</replica>
    </macros>
</yandex>
```

对于节点3：10.64.148.135   存储分片02 副本 01， shard=02  replica=c1-02-01(表示集群c1, 02分片, 01副本)

```
<?xml version="1.0" encoding="UTF-8"?>
<yandex>
    <macros>
        <shard>02</shard>
        <replica>c1-02-01</replica>
    </macros>
</yandex>
```

对于节点4：10.64.138.24   存储分片02 副本 02， shard=02 replica=c1-02-02(表示集群c1, 02分片, 02副本)

```
<?xml version="1.0" encoding="UTF-8"?>
<yandex>
    <macros>
        <shard>02</shard>
        <replica>c1-02-02</replica>
    </macros>
</yandex>
```

由上面配置可以看到replica的分布规律，其中shard表示分片编号；replica是副本标识，这里使用了c1-{shard}-{replica}的表示方式，比如c1-02-1表示c1集群的02分片下的1号副本，这样既非常直观的表示又唯一确定副本. 

### 集群配置 /data/clickhouse/conf/config.d/config.remote_servers.xml

这里指定了两个分片，每个分片两个副本。

```
<?xml version="1.0"?>
<yandex>
    <remote_servers replace="replace">
        <base>
            <shard>
                <weight>1</weight>
                <internal_replication>true</internal_replication>

                <replica>
                    <host>10.64.148.133</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>x</password>
                </replica>
                <replica>
                    <host>10.64.148.134</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>x</password>
                </replica>

            </shard>
            <shard>
                <weight>1</weight>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>10.64.148.135</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>x</password>
                </replica>
                <replica>
                    <host>10.68.138.24</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>x</password>
                </replica>

            </shard>
        </base>
    </remote_servers>
</yandex>
```

## 启动clickhouse-server进程

执行bash start_ch.sh start

## 测试

### 建表

```
create table t_sample3 on cluster base (id UInt64, its DateTime default now()) Engine=ReplicatedMergeTree('/clickhouse/tables/{shard}/table_name','{replica}') order by (its,id);

CREATE TABLE dis_sample3 on cluster base AS t_sample3  ENGINE = Distributed(base, default, t_sample3, rand())
```

插入数据：

```
在节点1执行

insert into dis_sample3 (id) select * from numbers(10000000);
结果为：
ip-10-64-148-133.yygamedev.com :) select count() from dis_sample3

SELECT count()
FROM dis_sample3

┌──count()─┐
│ 10000000 │
└──────────┘

1 rows in set. Elapsed: 0.146 sec.
```

查看每个节点part_log

节点1：

```
ip-10-64-148-133.yygamedev.com :) SELECT event_type, path_on_disk FROM system.part_log where table='t_sample3'

SELECT
    event_type,
    path_on_disk
FROM system.part_log
WHERE table = 't_sample3'

┌─event_type─┬─path_on_disk────────────────────────────────────────┐
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_0_0_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_1_1_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_2_2_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_3_3_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_4_4_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_5_5_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_6_6_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_7_7_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_8_8_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_9_9_0/ │
│ MergeParts │ /data1/clickhouse/data/default/t_sample3/all_0_5_1/ │
└────────────┴─────────────────────────────────────────────────────┘

11 rows in set. Elapsed: 0.019 sec.
```

节点2：

```
ip-10-64-148-134.yygamedev.com :) SELECT event_type, path_on_disk FROM system.part_log where table='t_sample3'

SELECT
    event_type,
    path_on_disk
FROM system.part_log
WHERE table = 't_sample3'

┌─event_type───┬─path_on_disk────────────────────────────────────────┐
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_0_0_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_1_1_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_2_2_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_3_3_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_4_4_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_5_5_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_6_6_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_7_7_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_8_8_0/ │
│ MergeParts   │ /data1/clickhouse/data/default/t_sample3/all_0_5_1/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_9_9_0/ │
└──────────────┴─────────────────────────────────────────────────────┘

11 rows in set. Elapsed: 0.003 sec.
```

节点3：

```
ip-10-64-148-135.yygamedev.com :) SELECT event_type, path_on_disk FROM system.part_log where table='t_sample3'

SELECT
    event_type,
    path_on_disk
FROM system.part_log
WHERE table = 't_sample3'

┌─event_type─┬─path_on_disk────────────────────────────────────────┐
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_0_0_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_1_1_0/ │
└────────────┴─────────────────────────────────────────────────────┘
┌─event_type─┬─path_on_disk────────────────────────────────────────┐
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_2_2_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_3_3_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_4_4_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_5_5_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_6_6_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_7_7_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_8_8_0/ │
│ NewPart    │ /data1/clickhouse/data/default/t_sample3/all_9_9_0/ │
│ MergeParts │ /data1/clickhouse/data/default/t_sample3/all_0_6_1/ │
└────────────┴─────────────────────────────────────────────────────┘
```

节点4

```
ip-10-64-138-24.yygamedev.com :) SELECT event_type, path_on_disk FROM system.part_log where table='t_sample3'

SELECT
    event_type,
    path_on_disk
FROM system.part_log
WHERE table = 't_sample3'

┌─event_type───┬─path_on_disk────────────────────────────────────────┐
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_0_0_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_1_1_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_2_2_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_3_3_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_4_4_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_5_5_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_6_6_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_7_7_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_8_8_0/ │
│ DownloadPart │ /data1/clickhouse/data/default/t_sample3/all_9_9_0/ │
│ MergeParts   │ /data1/clickhouse/data/default/t_sample3/all_0_6_1/ │
└──────────────┴─────────────────────────────────────────────────────┘

11 rows in set. Elapsed: 0.003 sec.
```

可以看到，节点2和节点4都是DownloadPart



## 停掉一个节点，对查询没有影响