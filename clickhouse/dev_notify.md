# ck集群运维注意事项

{:toc}
* TOC

1. alter时，只需要在一个分片上执行即可
2. 遇到zookeeper session expire删除不了表，可以先detach再attach
3. ck mutation可以保证异步操作不会删除新写入的数据，可以通过system.mutations查看进度
4. 多磁盘空出两块盘，单独部署zk,并且将zk的log和snapshot分开
5. 遇到too many parts时，需要控制写入速度。sinker看看是不是负载没有好,可以 alter table t_dw_huya_page_pv_ck on cluster single modify setting parts_to_throw_insert=600 增大抛异常的值，但是本质上还是后台merge跟不上插入，可以增大io性能，或减少分区
6. 没有副本的cluster,要设置 <internal_replication>false</internal_replication>
7. 修改```<background_pool_size>64</background_pool_size> 
      <background_schedule_pool_size>64</background_schedule_pool_size>
      <background_move_pool_size>32</background_move_pool_size>```调大后台执行线程数
8. 修改```<max_partitions_per_insert_block>1000</max_partitions_per_insert_block>``` 解决 DB::Exception: Too many partitions for single INSERT block (more than 100)的问题
9. 修改```<max_replicated_merges_in_queue>200</max_replicated_merges_in_queue>
        <max_replicated_mutations_in_queue>200</max_replicated_mutations_in_queue>```
解决  Number of queued merges (36) and part mutations (0) is greater than max_replicated_merges_in_queue (16)  的错误
11. 查看集群事件和指标
SELECT 
    event_time,
    ProfileEvent_InsertQuery,
    runningDifference(ProfileEvent_InsertQuery) ins_per_sec
FROM system.metric_log
WHERE event_date = today()
ORDER BY event_time DESC
LIMIT 50;
SELECT value
FROM system.events
WHERE event LIKE 'InsertQuery'

12. 查看多少mutation没执行```select table,command,mutation_id,create_time,is_done,latest_fail_time,latest_fail_reason,parts_to_do from system.mutations where table= 't_oexp_precomputation_all_metric' order by create_time limit  10```
13. 碰到 The local set of parts of table default.dwd_test doesn’t look like the set of parts in ZooKeeper 的问题，一般都是手动操作了表的ddl导致不同步，可以把metadata下的这个表先移除，让server可以正常启动，然后再重建表，数据会自动同步
14.  select count() as c ,database||'.'||table from system.replication_queue where type='MUTATE_PART'  group by database,table order by c
15.  select count() as c ,database||'.'||table from system.mutations where is_done=0 group by database,table order by c
16.  遇到卡住的时候，可以先把卡住的表detach，然后批次attach回来