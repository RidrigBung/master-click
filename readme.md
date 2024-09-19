volumes:
  - ${storage_path}/clickhouse_node_01:/etc/clickhouse-server/

storage_path=/mnt/c/Users/Andrew/repository/master-click/clickhouse_storage

volumes:
  - ${storage_path}/clickhouse_node_04:/etc/clickhouse-server/

docker volume create my-vol

> cd ./docker
<!-- > docker-compose -f .\docker-compose-init.yaml up -d -->
> docker-compose up -d
add config.xml to folers inside clickhouse_storage 
