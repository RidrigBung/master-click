Финальный проект-5. 
Развернуть в docker compose 4х кликов в шардово-реплицируемой схеме. 2 шарда, 2 реплики на каждый шард. 1 clickhouse-keeper. И еще один главный клик на котором будут таблицы distributed указывающие на шардовые таблицы.

1. Пишем docker-compose файл на 5 нод клика, 4 клика для шардово-реплицируемой схемы, первая из которых будет иметь настройку clickhouse-keeper, а пятый клик будет  содержать distributed таблицы.

2. Запускаем docker-compose файл:
> cd ./docker
> docker-compose up -d

3. Настроим ноды кластера:
Заходим в `/etc/clickhouse-server/config.xml` ноды 1, 2, 3 и вставляем настройки из <src="./configs/merged_config_keeper.xml">merged_config_keeper</src>, заменяя `${SERVER_ID}`, `${SHARD}`, `${REPLICA}` на соответствующие номер сервера, шарда и реплики, для ноды 4 вставляем только <src="./configs/merged_config.xml">merged_config</src>.
Не забудем также добавить в master_clickhouse config файл добавить <src="./configs/merged_config_master.xml">merged_config_master</src>.

4. Работа с нодами:
Подключимся к первой ноде по адресу localhost:8123 с помощью интерфейса DBeaver и выполним следующие запросы, чтобы убедиться в корректности кластера:
`
select *
from system.zookeeper
where path = '/'
format Vertical
;
`
`
select cluster
	  , host_name
	  , host_address
	  , port
	  , shard_num
	  , replica_num
from system.c
lusters
;
`

5. Настроим кластер:
Создадим db и реплицированную таблицу на кластере.
`
CREATE DATABASE test ON CLUSTER 'cluster';
`
`
CREATE TABLE test.shk_uuid ON CLUSTER '{cluster}' (
	shk_id Int64,
	dt DateTime
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/{table}', '{replica}')
PARTITION BY toYYYYMM(dt)
ORDER BY (shk_id)
;
`

6. Настроим main:
Подключаемся к master_clickhouse_log, создадим distributed таблицу на master_clickhouse и вставим туда тестовые данные:
`
create table default.shk_uuid (
	shk_id Int64,
	dt DateTime
)
ENGINE = Distributed('cluster', test, shk_uuid, shk_id % 2)
;
`
`
insert into default.shk_uuid 
select * from generateRandom('shk_id Int64, dt DateTime', 1, 10, 2) limit 100
;
`

7. Проверим вставленные данные:
`
select * from default.shk_uuid;
select count() from default.shk_uuid;
`
В каждой ноде есть хотя бы одна запись:
`
select *
from
(
    select
        hostName(),
        *
    from remote('172.18.0.4', 'test', 'shk_uuid', 'default', 'default')
    limit 1
    union all
    select
        hostName(),
        *
    from remote('172.18.0.5', 'test', 'shk_uuid', 'default', 'default')
    limit 1
    union all
    select
        hostName(),
        *
    from remote('172.18.0.2', 'test', 'shk_uuid', 'default', 'default')
    limit 1
    union all
    select
        hostName(),
        *
    from remote('172.18.0.6', 'test', 'shk_uuid', 'default', 'default')
    limit 1
)
;
`
В запросе ниже видно, что кол-во записей распределилось почти равномерно между двумя шардами, а также количество записей в репликах одного шарда равно
`
select node
	 , count() qty
from
(
    select
    	'node01' node,
        *
    from remote('172.18.0.4', 'test', 'shk_uuid', 'default', 'default')
    union all
    select
    	'node02' node,
        *
    from remote('172.18.0.5', 'test', 'shk_uuid', 'default', 'default')
    union all
    select
    	'node03' node,
        *
    from remote('172.18.0.2', 'test', 'shk_uuid', 'default', 'default')
    union all
    select
    	'node04' node,
        *
    from remote('172.18.0.6', 'test', 'shk_uuid', 'default', 'default')
)
group by node
order by node
;
`
P.S. Metadata in zookeeper is deleted async, so you need wait for 5~10 min then can success. Or delete Metadata sync:
`drop table test.test_local sync;`