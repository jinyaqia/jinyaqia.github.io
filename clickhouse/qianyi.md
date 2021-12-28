{:toc}
# 背景

现有的ck集群没有副本，磁盘是12块盘的stat盘，存在磁盘故障导致数据丢失的风险，而依赖zk的双副本策略又由于zk性能存在瓶颈影响集群的可用性，故而使用带三副本的高效云盘替代stat盘，规避数据丢失的风险。

当前ck的写入程序使用的是统一的域名，由域名查询到对应的ip节点来建立tcp连接。对于ck的查询，使用的是内部的一个代理，该代理配置了集群的ip，由代理去轮询ip进行查询。

在数据迁移的过程中，需要保证集群写入和查询都不受影响，关键在于控制好查询和写入的节点。

# 迁移方案

## 1. 新节点安装ck，同步元数据，元数据管理覆盖新节点ip

新节点安装时，配置文件的remote_servers中的集群配置和旧集群一致，新节点只是做一个查询转发。

从旧节点中导出非system表的建表语句，由于一份表对应着一个本地表和一个分布式表，因此需要先创建完本地表。

```
导出本地表
echo "select create_table_query||';'  from system.tables where database != 'system' and engine!='Distributed' order by name desc" | /data/clickhouse/bin/clickhouse-client --password xxx --port 9000 > localtable.txt

导出分布式表的ddl
echo "select create_table_query||';'  from system.tables where database != 'system' and engine='Distributed' order by name desc" | /data/clickhouse/bin/clickhouse-client --password xxx --port 9000 > distable.txt

```

将建表语句发送到新节点执行

```
执行本地表的ddl
./sendfile.sh ckip.txt localtable.txt distable.txt
./runcmd.sh ckip.txt "/data/clickhouse/bin/clickhouse-client  --password xxx -mn < /home/jinyaqia/ck_tool/localtable.txt"

执行分布式表的ddl
./runcmd.sh ckip.txt "/data/clickhouse/bin/clickhouse-client  --password xxx -mn < /home/jinyaqia/ck_tool/distable.txt"
```

其中

```
# sendfile.sh
# 用于发送文件
#!/usr/bin/env bash

file=''
array="$@"
i=0
for el in $array
do
    if [ ${i} -ne 0 ]
	then
		file=$file' '${el}
    fi
	let i++
done

for ip in $(cat $1|grep -v "^#.*\|^$")
do
    cmd="scp ${file} $ip:~/ck_tool/"
	$cmd &
	echo $ip,$cmd
done

wait
echo 'done'
```

```
# runcmd.sh
# 用于执行命令
#!/usr/bin/env bash

for ip in $(cat $1|grep -v "^#.*\|^$")
do
	echo $ip
    ssh $ip "${2}" &
    echo "end ${ip}"
done

wait
```

## 2. 配置查询代理，从新节点查询数据
## 3. 滚动迁移
### 3.1 先停掉待迁移节点的写入
域名dns管理去掉待迁移节点，这里需要写入端没有直接缓存ck节点ip，确保在数据复制的过程中，新的数据不会写入待迁移的节点
### 3.2 ck节点一对一复制数据
采用clickhouse copier的方式进行迁移，生成所有本地表的迁移配置，调用迁移脚本进行数据复制

1. 导出需要进行迁移的表，可以从system.tables中查询并导出

```
echo "select database||'.'||name,engine_full  from system.tables where database != 'system' and engine not in ('Distributed','View','MaterializedView')  order by name desc" | /data/clickhouse/bin/clickhouse-client --password xxx --port 9000 > dbtable.txt.all

```

导出的文件去掉.inner.的表，物化视图等普通本地表导完后再处理
如果有转义的,如'\，手工替换掉就可以了

2. 将脚本和文件都上传到接口机，由接口机发送到新节点,脚本内容后文给出

```
dbtable.txt.all  需要迁移的表
dbtable.txt.todo 需要执行迁移的表
table_copy_copier.py  迁移脚本
table_check.py  数据一致性检查脚本
get_error.py  处理迁移异常的表
ipmap.txt     旧节点复制到新节点的ip对应关系
copier_template.xml 复制工具配置模板
zk.xml  复制工具zk配置

发送命令
./sendfile.sh ckip.txt dbtable.txt.all dbtable.txt.todo table_copy_copier.py table_check.py get_error.py copier_template.xml zk.xml
```

3. 执行复制

```
nohup python3 table_copy_copier.py 2>&1 >copy.log &
```

4. 校对、检查和重跑

```
批量校对
python3 table_check.py
python3 get_error.py

有异常的表会写入dbtable.txt.1
其中新节点数据条数少的写入error.txt.less
新节点数据条数多了的写入error.txt.more

手动核对下dbtable.txt.1里的表迁移异常的原因，可以查clickhouse-copier的日志，如果要重跑，则生成新的dbtable.txt.todo, 删掉zk中对应表的节点，然后调用table_copy_copier.py重新复制

```
### 3.3 更新ck集群配置remote_servers，增加新节点，去掉旧节点，等待ck server自动加载更新配置，此时查询会落到新节点上

### 3.4 配置域名，使数据写入新节点

## 4. 退回机器

# 注意点

1. 该方案在数据复制的时候要求旧节点和新节点都不会写数据，这个需要写入的时候不写分布式表
2. 复制任务重跑时，需要删掉zk节点的数据
3. 由于复制任务会在zk上创建比较多的节点数，为了降低zk的延迟，不要用ck集群的zk，可以单独搭建一个zk集群
4. table_copy_copier.py调用 clickhouse copier命令后，即使主进程关闭了，copier任务也会一直运行
5. table_copy_copier.py配置了60个子线程用于调用copier任务，如果要降低带宽的使用，可以调低并行度


# 附录：
## table_copy_copier.py

```python
import sys,json;
import socket;
import os;
from concurrent.futures import ThreadPoolExecutor,as_completed
import queue,threading;
import time,functools

def get_time():
    time_now=int(time.time())
    time_local=time.localtime(time_now)
    return time.strftime("%Y-%m-%d %H:%M:%S",time_local)
   
def exec_command(fo,db,table,command_template,ip,task_file):
    try:
        task_path='/copytask/{}/{}/{}'.format(ip,db,table)
        base_dir=table
        if not os.path.exists(base_dir):
            os.makedirs(base_dir)
        command=command_template.format(task_path=task_path,base_dir=base_dir,task_file=task_file)
        print("start thread num={} command={} pid={}".format(threading.get_ident(),command,os.getpid()))
        f = os.popen(command)
        print("done thread num={} command={} pid={}".format(threading.get_ident(),command,os.getpid()))
        res="start {}.{} time={}\n".format(db,table,get_time())
        fo.write(res)
        fo.flush()
    except:
        error="start {}.{} error\n".format(db,table)
        fo.write(error)
        fo.flush()
    return f

def read_db_table(file_name):
    try:
        f = open(file_name)
        lines = f.read().split('\n')
    finally:
        f.close()
    return lines

def read_file(file_name):
    try:
        f = open(file_name)
        lines = f.read()
    finally:
        f.close()
    return lines

def write_file(file_name,content):
    try:
        f = open(file_name,"w+")
        f.write(content)
    finally:
        f.close()

def get_local_ip():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(('8.8.8.8', 80))
        localIp = s.getsockname()[0]
    finally:
        s.close()
    return localIp

def task_done_callback(future,dbtable_str):
    try:
        data = future.result()
        res="done {} time {}\n".format(dbtable_str,get_time())
    except CancelledError:
        res="error {} time {}\n".format(dbtable_str,get_time())
    finally:
        fo = open("result.txt","a+")
        fo.write(res)
        fo.flush()
        print('result',data.read())
        fo.close()


if __name__ == '__main__':

    ipmap=json.loads(read_file("ipmap.txt"))
    print(ipmap)
    localIp = get_local_ip()
    print(localIp)
    remoteIp = ipmap[localIp]
    print(remoteIp)
    # copier template
    copier_template = read_file("copier_template.xml")
    print(copier_template)
    # db.table needs to copy
    lines = filter(None,read_db_table("dbtable.txt.todo"))
    print(lines)
    command_template="/data/clickhouse/bin/clickhouse copier --config zk.xml --task-path {task_path} --task-file {task_file} --base-dir {base_dir}"
    result_file = open("task_log.txt","a+")
    pool = ThreadPoolExecutor(60)
    waitQueue = queue.Queue()
    for line in lines:
        waitQueue.put(line)
    
    while(waitQueue.qsize()>0):
        line_arr = waitQueue.get().split('\t')
        com=line_arr[0]
        task_file="task_{}.xml".format(com)
        dbtable=com.split(".")
        content=copier_template.format(source_host=remoteIp,target_host=localIp,database=dbtable[0],table=dbtable[1],engine=line_arr[1])
        write_file(task_file,content)
        task = pool.submit(exec_command, result_file, dbtable[0],dbtable[1], command_template, remoteIp, task_file).add_done_callback(functools.partial(task_done_callback,dbtable_str=com))
        
```

## table_check.py

```
import sys,json;
import socket;
import os;
from concurrent.futures import ThreadPoolExecutor,as_completed,CancelledError
import queue,threading;
import time,functools

def get_time():
    time_now=int(time.time())
    time_local=time.localtime(time_now)
    return time.strftime("%Y-%m-%d %H:%M:%S",time_local)
   
def exec_command(fo, dbtable_str,sql):
    
    try:        
        dbtable=dbtable_str.split(".")
        command=sql.format(db=dbtable[0],table=dbtable[1])
        print("start thread num={} command={} pid={}".format(threading.get_ident(),command,os.getpid()))
        f = os.popen(command)
        res="start {} time={}\n".format(dbtable_str,get_time())
        fo.write(res)
        fo.flush()
    except:
        error="start {} error\n".format(dbtable_str)
        fo.write(error)
        fo.flush()
    return f

def read_db_table(file_name):
    try:
        f = open(file_name)
        lines = f.read().split('\n')
    finally:
        f.close()
    return lines

def read_file(file_name):
    try:
        f = open(file_name)
        lines = f.read()
    finally:
        f.close()
    return lines

def get_local_ip():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(('8.8.8.8', 80))
        localIp = s.getsockname()[0]
    finally:
        s.close()
    return localIp

def task_done_callback(future,fo, dbtable_str,engine):
    res=""
    try:
        data = future.result()
        counts = data.read().split('\n')
        line1= counts[0].split('\t')
        line2=counts[1].split('\t')
        count1=count2=0
        if(line1[0]=='a'):
            count1=line1[1]
            count2=line2[1]
        else:
            count1=line2[1]
            count2=line1[1]
        res="done|{}\t{}|result@{}@{}@{}\n".format(dbtable_str,engine,count1,count2,int(count1)-int(count2))
    except CancelledError:
        res="error {} \n".format(dbtable_str)
    finally:
        fo.write(res)
        fo.flush()
        print('result',res)
    


if __name__ == '__main__':
    ipmap=json.loads(read_file("ipmap.txt"))
    print(ipmap)
    localIp = get_local_ip()
    print(localIp)
    remoteIp = ipmap[localIp]
    print(remoteIp)
    sql="echo \"select 'a',count() from remote('"+remoteIp+"','{db}','{table}','default','IARYxRcr') union all select 'b',count() from {db}.{table};\" |/data/clickhouse/bin/clickhouse-client  --password IARYxRcr"
    lines = filter(None,read_db_table("dbtable.txt.all"))
    print(lines)
    fo = open("checkresult.txt","w+")
    fo1 = open("checkresult1.txt","w+")
    pool = ThreadPoolExecutor(60)
    waitQueue = queue.Queue()
    for line in lines:
        waitQueue.put(line)
    
    while(waitQueue.qsize()>0):
        line_arr = waitQueue.get().split('\t')
        com=line_arr[0]
        engine=line_arr[1]
        task = pool.submit(exec_command, fo, com, sql).add_done_callback(functools.partial(task_done_callback,dbtable_str=com,fo=fo1,engine=engine))
        
```

## get_error.py

```
import sys
import json
import socket
import os
from concurrent.futures import ThreadPoolExecutor, as_completed
import queue
import threading
import time
import functools


def read_file(file_name):
    try:
        f = open(file_name)
        lines = f.read().split('\n')
    finally:
        f.close()
    return lines


if __name__ == '__main__':
    fo = open("dbtable.txt.1", "w+")
    done = open('finish.txt', "w+")
    lines = read_file("checkresult1.txt")
    # 比旧节点少了
    err_less = open('error.txt.less', 'w+')
    # 比旧节点多了
    err_more = open('error.txt.more', 'w+')
    todo = 0
    to_delete = 0
    finish = 0
    for line in lines:
        if(line != ''):
            line_arr = line.split('|')
            print(line_arr)
            result = line_arr[2].split('@')
            dbtable = line_arr[1].split('\t')[0]
            old=int(result[1])
            new = int(result[2])
            diff=int(result[3])
    
            if(diff != 0 and new == 0):
                todo += 1
                fo.write(line_arr[1]+'\n')
            elif(diff != 0 and new != 0):
                todo += 1
                to_delete += 1
                fo.write(line_arr[1]+'\n')
            else:
                finish += 1
                done.write(dbtable+'\n')

            if(diff>0):
                err_less.write("{} {}\n".format(dbtable,result))
            elif(diff<0):
                err_more.write("{} {}\n".format(dbtable,result))

    print("finish={}, todo={}".format(finish, to_delete, todo))
    fo.close()
    done.close()
    err_less.close()
    err_more.close()

```

## copier_template.xml

```
<yandex>
    <remote_servers>
        <source_cluster>
            <shard>
                <internal_replication>false</internal_replication>
                    <replica>
                        <host>{source_host}</host>
                        <port>9000</port>
                        <user>default</user>
                        <password>IARYxRcr</password>
                        <secure>0</secure>
                    </replica>
            </shard>
        </source_cluster>

        <destination_cluster>
            <shard>
                <internal_replication>false</internal_replication>
                    <replica>
                        <host>{target_host}</host>
                        <port>9000</port>
                        <user>default</user>
                        <password>IARYxRcr</password>
                        <secure>0</secure>
                    </replica>
            </shard>89
        </destination_cluster>
    </remote_servers>

    <max_workers>32</max_workers>

    <!-- Setting used to fetch (pull) data from source cluster tables -->
    <settings_pull>
        <readonly>1</readonly>
    </settings_pull>

    <settings_push>
        <readonly>0</readonly>
    </settings_push>

     <settings>
        <connect_timeout>3</connect_timeout>
         <insert_distributed_sync>1</insert_distributed_sync>
    </settings>

    <tables>
        <table_hits>
            <!-- Source cluster name (from <remote_servers/> section) and tables in it that should be copied -->
            <cluster_pull>source_cluster</cluster_pull>
            <database_pull>{database}</database_pull>
            <table_pull>{table}</table_pull>

            <!-- Destination cluster name and tables in which the data should be inserted -->
            <cluster_push>destination_cluster</cluster_push>
            <database_push>{database}</database_push>
            <table_push>{table}</table_push>

            <engine>
            ENGINE={engine}
            </engine>
            <sharding_key>rand()</sharding_key>

        </table_hits>
    </tables>
</yandex>
```

## ipmap.txt

```
{
    "10.69.28.1": "10.69.29.11",
    "10.69.29.2": "10.69.28.22",
    "10.69.29.3": "10.69.29.33",
    "10.69.29.4": "10.69.29.44"
}
```

## zk.txt
```
<yandex>
    <logger>
        <level>trace</level>
        <size>100M</size>
        <count>3</count>
    </logger>
    <zookeeper>
        <node index="0">
            <host>zkhost</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>
```