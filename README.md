# ДЗ: Развернуть HA кластер

## Вариант №1: Cluster control

Для выполнения данного варианта развернул в GKE три инстанса:
- instance-2 (10.128.0.3)
- instance-3 (10.128.0.4)
- instance-4 (10.128.0.5)

На instance-2 развернул ClusterControl через docker:
```
yum install docker

docker pull severalnines/clustercontrol

docker run -d -e DOCKER_HOST_ADDRESS="192.168.10.10" -p 5000:80 -p 5001:443 severalnines/clustercontrol
```

Тут немного пришлось помучиться с запуском, т.к. без DOCKER_HOST_ADDRESS контейнер корректно стартовать не хотел, а в доке по установке этого нет.

Далее после регистрации я раскидал по серверам instance-3 и instance-4 ssh-ключи и начал настройку кластера в интерфейсе ClusterControl. Установил instance-3 (10.128.0.4) в качестве мастера и instance-4 (10.128.0.5) в качестве асинхронного слейва.

После успешного завершения настройки кластер появился в интерфейсе. Также поставил QueryMonitor, посмотрел, как он отображает информацию о выполняемых на серверах запросах. В целом интерфейс показался довольно удобным и приятным)

## Вариант №2: pg_auto_failover

Для выполнения данного варианта развернул в GKE три инстанса:
- instance-5 (10.138.0.2)
- instance-6 (10.138.0.3)
- instance-7 (10.138.0.4)

На сервера instance-5 и instance-6 установил postgresql-11.  
На instance-7 устанавливал и настраивал pg_auto_failover:
```
curl https://install.citusdata.com/community/rpm.sh | sudo bash

yum install -y pg-auto-failover10_11

sudo su - postgres
export PATH="$PATH:/usr/pgsql-11/bin"

pg_autoctl create monitor --nodename 10.138.0.4 --pgdata ./monitor --pgport 6000 --auth trust
```
Далее настраиваю мастера на instance-5:
```
curl https://install.citusdata.com/community/rpm.sh | sudo bash

yum install -y pg-auto-failover10_11

sudo su - postgres

export PATH="$PATH:/usr/pgsql-11/bin"

pg_autoctl create postgres     \
    --pgdata /var/lib/pgsql/11/data \
    --auth trust \
    --dbname otusdb \
    --nodename 10.138.0.2      \
    --pgctl /usr/pgsql-11/bin/pg_ctl \
    --monitor postgres://autoctl_node@10.138.0.4:6000/pg_auto_failover
```
В ходе выполнения pg_autoctl встретил пару проблем:
- Сначала команда "pg_autoctl create postgres..." не выполнялась из-за настроек og_hba БД монитора, после правки pg_hba от этой ошибки я исбавился.
- Далее pg_autoctl ругался на тот postgres, что был уже установлен, т.к. занимал указанный мною каталог с pgdata. Остановил postgres, удалил каталог и команды выполнилась успешно.
Далее аналогичные настройки на instance-6:
```
curl https://install.citusdata.com/community/rpm.sh | sudo bash

yum install -y pg-auto-failover10_11

sudo su - postgres

export PATH="$PATH:/usr/pgsql-11/bin"

pg_autoctl create postgres     \
    --pgdata /var/lib/pgsql/11/data \
    --auth trust \
    --dbname otusdb \
    --nodename 10.138.0.3      \
    --pgctl /usr/pgsql-11/bin/pg_ctl \
    --monitor postgres://autoctl_node@10.138.0.4:6000/pg_auto_failover
```

После выполнения данных действий на instance-7 можно наблюдать текущее состояние репликации:
```
      Name |   Port | Group |  Node |     Current State |    Assigned State
-----------+--------+-------+-------+-------------------+------------------
10.138.0.2 |   5432 |     0 |     2 |           primary |           primary
10.138.0.3 |   5432 |     0 |     3 |         secondary |         secondary
```

Также можем проверить непосредственно работу репликации.  
```
-- На сервере isntance-5:
create table test(i int, t text);

-- На сервере instance-6:
otusdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

-- На сервере isntance-5:
insert into test values(1, '1'), (2, '2'), (3, '3');

-- На сервере instance-6:
otusdb=# select * from test;
 i | t
---+---
 1 | 1
 2 | 2
 3 | 3
(3 rows)
```

## Вариант №3: Pacemaker + corosync + drbd

Для выполнения данного варианта развернул в GKE два инстанса:
- instance-11 (10.166.0.5, 172.16.0.2)
- instance-12 (10.166.0.7, 172.16.0.3)

Сначала произведем необходимые настройки на ВМ instance-11 и instance-12.  
Вылючаем SELinux:
```
vi /etc/selinux/config

SELINUX=disabled
```
Прописываем в etc-hosts оба сервера:
```
cat /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.0.3 instance-12.europe-north1-a.c.postgres2021-26091996-338403.internal instance-12
172.16.0.2 instance-11.europe-north1-a.c.postgres2021-26091996-338403.internal instance-11  # Added by Google
169.254.169.254 metadata.google.internal  # Added by Google
```
Устанавливаем drbd и перезагружаем ВМ:
```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -ivh http://elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
yum install -y drbd90-utils kmod-drbd90
reboot
```
Приводим файл /etc/drbd.conf к виду:
```
include "drbd.d/global_common.conf";
include "drbd.d/*.res";
```
Приводим файл /etc/drbd.d/global_common.conf к виду (по факту строк там больше, но они закомментированы, поэтому тут я их решил не отражать):
```
global {
 usage-count no;
}
common {
 net {
  protocol C;
 }
}
```
Приводим файл /etc/drbd.d/postgres.res к виду:
```
resource postgres {
startup {
}
disk { on-io-error detach; }
device      /dev/drbd0;
disk        /dev/sdb;

on instance-11 {
address     10.166.0.5:7788;
meta-disk   internal;
node-id     0;
}

on instance-12 {
address     10.166.0.7:7788;
meta-disk   internal;
node-id     1;
}
}
```
Добавляем правила в firewalld (хотя в данном случае это не обязательно):
```
firewall-cmd --add-port=7788/tcp --permanent
firewall-cmd --reload
```
Создаем ресурс DRBD и стартуем сервис:
```
drbdadm create-md postgres
drbdadm up postgres
systemctl start drbd
```
После этого результат настроек можно посмотреть в lsblk:
```
[root@instance-11 ~]# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0   20G  0 disk
├─sda1    8:1    0  200M  0 part /boot/efi
└─sda2    8:2    0 19.8G  0 part /
sdb       8:16   0  100G  0 disk
└─drbd0 147:0    0  100G  0 disk
```
Т.к. по умолчанию DRBD стартует в статусе secondary, меняем статус на instance-11:
```
drbdadm primary postgres --force
```
Текущий статус DRBD удобно смотреть командой:
```
[root@instance-11 ~]# drbdsetup status -vs
postgres node-id:0 role:Primary suspended:no
    write-ordering:flush
  volume:0 minor:0 disk:UpToDate backing_dev:/dev/sdb quorum:yes
      size:104854364 read:24378184 written:0 al-writes:0 bm-writes:1069
      upper-pending:0 lower-pending:4 al-suspended:no blocked:no
  instance-12 node-id:1 connection:Connected role:Secondary congested:no
      ap-in-flight:0 rs-in-flight:232
    volume:0 replication:SyncSource peer-disk:Inconsistent done:23.25
        resync-suspended:no
        received:0 sent:24378184 out-of-sync:80476296 pending:5 unacked:4
        dbdt1:96.88 eta:811
```
Далее останавливаем drbd и отключаем автозапуск на обоих серверах и затем устанавливаем postgres:
```
systemctl stop drbd
systemctl disable drbd

yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

yum install -y postgresql13 postgresql13-server
```
Дальнейшие действия выполняем только на instance-11. Нужно занести директорию /var/lib/pgsql на drbd:
```
/usr/pgsql-13/bin/postgresql-13-setup initdb

mkdir /root/backup
mv /var/lib/pgsql/* /root/backup/

systemctl start drbd
drbdadm primary postgres 

mount -t xfs /dev/drbd0 /var/lib/pgsql/
mv /root/backup/* /var/lib/pgsql/
chown -R postgres:postgres /var/lib/pgsql/
```
Теперь вносим изменения в конфиги. В postgresql.conf устанавливаем следующие параметры:
```
listen_addresses = '*'
wal_level = hot_standby
synchronous_commit = local
archive_mode = on
archive_command = 'cp %p /var/lib/pgsql/13/data/archive/%f'
```
Добавляем в pg_hba строку:
```
host    all             all             0.0.0.0/0               trust
```
И запускам postgres:
```
service postgresql-13 start
```
Запишем в БД немного данных для теста:
```
su - postgres
psql

postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# create table test(i int, t text);
CREATE TABLE
otus=# insert into test values(1, '1'), (2, '2'), (3, '3');
INSERT 0 3
```
Ставим pacemaker на обе ноды:
```
yum install pcs pacemaker corosync fence-agents-virsh fence-virt \
pacemaker-remote fence-agents-all lvm2-cluster resource-agents \
psmisc policycoreutils-python gfs2-utils

echo "otus" | passwd hacluster --stdin

systemctl start pcsd.service; systemctl enable pcsd.service
```
Продолжаем настройку pacemaker на instance-11:
```
pcs cluster auth instance-11 instance-12 -u hacluster -p otus

pcs cluster setup --start --name otus-hw instance-11 instance-12

pcs cluster enable --all
```
Проверяем текущее состояние кластера:
```
pcs status

Cluster name: otus-hw

WARNINGS:
No stonith devices and stonith-enabled is not false

Stack: corosync
Current DC: instance-12 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Sun Jan 23 11:27:34 2022
Last change: Sun Jan 23 11:22:53 2022 by hacluster via crmd on instance-12

2 nodes configured
0 resource instances configured

Online: [ instance-11 instance-12 ]

No resources


Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```
Отключаем stonith и quorum-policy и добавляем ресурсы:
```
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore

pcs cluster cib clust_cfg
pcs -f clust_cfg property set no-quorum-policy=ignore
pcs -f clust_cfg resource defaults resource-stickiness=100
pcs -f clust_cfg resource create drbd_postgres ocf:linbit:drbd drbd_resource=postgres op monitor interval=30s
pcs -f clust_cfg resource master drbd_master_slave drbd_postgres master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
pcs -f clust_cfg resource create postgres_fs ocf:heartbeat:Filesystem device=/dev/drbd0 directory=/var/lib/pgsql fstype=xfs
pcs -f clust_cfg resource create postgresql_service ocf:heartbeat:pgsql op monitor timeout=30s interval=30s
pcs -f clust_cfg resource create postgres_vip ocf:heartbeat:IPaddr2 ip="192.168.5.23" iflabel="pgvip" op monitor interval=30s
pcs -f clust_cfg resource group add postgres-group postgres_fs postgres_vip postgresql_service
pcs -f clust_cfg constraint colocation add postgres-group with drbd_master_slave INFINITY with-rsc-role=Master
pcs -f clust_cfg constraint order promote drbd_master_slave then start postgres-group

pcs -f clust_cfg constraint
pcs cluster cib-push clust_cfg
crm_verify -L
```
Смотрим, что получилось по итогу:
```
[root@instance-12 ~]# pcs status
Cluster name: otus-hw
Stack: corosync
Current DC: instance-12 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Sun Jan 23 11:50:44 2022
Last change: Sun Jan 23 11:33:48 2022 by root via cibadmin on instance-11

2 nodes configured
5 resource instances configured

Online: [ instance-11 instance-12 ]

Full list of resources:

 Master/Slave Set: drbd_master_slave [drbd_postgres]
     Masters: [ instance-12 ]
     Slaves: [ instance-11 ]
 Resource Group: postgres-group
     postgres_fs        (ocf::heartbeat:Filesystem):    Started instance-12
     postgres_vip       (ocf::heartbeat:IPaddr2):       Stopped
     postgresql_service (ocf::heartbeat:pgsql): Stopped

Failed Resource Actions:
* postgres_vip_start_0 on instance-11 'unknown error' (1): call=20, status=complete, exitreason='[findif] failed',
    last-rc-change='Sun Jan 23 11:33:49 2022', queued=0ms, exec=72ms
* postgres_vip_start_0 on instance-12 'unknown error' (1): call=28, status=complete, exitreason='[findif] failed',
    last-rc-change='Sun Jan 23 11:33:51 2022', queued=0ms, exec=69ms

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```
Да, и тут я понял, что забыл добавить на сервера интерфейс с разделяемым ip...
В целом, настройку данной схемы я примерно усвоил, но позже, думаю, все равно вернусь и добью этот вариант)
