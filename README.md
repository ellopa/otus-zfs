# Практические навыки работы с ZFS

## Определяем алгоритм с наилучшим сжатием

- Смотрим список всех дисков, которые есть в виртуальной машине: lsblk

```
elena_leb@ubuntunbleb:~/ZFS_DZ$ vagrant ssh
[vagrant@zfs ~]$ sudo -i
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk
```

- Создаём 4 пула из двух дисков в режиме RAID 1:

```
[root@zfs ~]# zpool create otus1 mirror /dev/sdb /dev/sdc
[root@zfs ~]# zpool create otus2 mirror /dev/sdd /dev/sde
[root@zfs ~]# zpool create otus3 mirror /dev/sdf /dev/sdg
[root@zfs ~]# zpool create otus4 mirror /dev/sdh /dev/sdi
```

- Смотрим информацию о пулах: zpool list

Команда zpool status показывает информацию о каждом диске, состоянии сканирования и об ошибках чтения, записи и совпадения хэш-сумм. 
Команда zpool list показывает информацию о размере пула, количеству занятого и свободного места, дедупликации и т.д.


```
[root@zfs ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```

- Добавим разные алгоритмы сжатия в каждую файловую систему:

    • Алгоритм lzjb: 
      zfs set compression=lzjb otus1
    • Алгоритм lz4: 
      zfs set compression=lz4 otus2
    • Алгоритм gzip: 
      zfs set compression=gzip-9 otus3
    • Алгоритм zle:  
      zfs set compression=zle otus4

```
[root@zfs ~]# zfs set compression=lzjb otus1
[root@zfs ~]# zfs set compression=lz4 otus2
[root@zfs ~]# zfs set compression=gzip-9 otus3
[root@zfs ~]# zfs set compression=zle otus4
```
- Проверим, что все файловые системы имеют разные методы сжатия:

```
[root@zfs ~]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
> Сжатие файлов будет работать только с файлами, которые были добавлены после включение настройки сжатия.

- Скачаем один и тот же текстовый файл во все пулы:

```
[root@zfs ~]# for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
--2023-12-03 12:51:10--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40997929 (39M) [text/plain]
Saving to: '/otus1/pg2600.converter.log'

100%[=====================================================================>] 40,997,929  4.36MB/s   in 17s    

2023-12-03 12:51:29 (2.27 MB/s) - '/otus1/pg2600.converter.log' saved [40997929/40997929]

--2023-12-03 12:51:29--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40997929 (39M) [text/plain]
Saving to: '/otus2/pg2600.converter.log'

100%[=====================================================================>] 40,997,929  4.39MB/s   in 8.0s   

2023-12-03 12:51:38 (4.89 MB/s) - '/otus2/pg2600.converter.log' saved [40997929/40997929]

--2023-12-03 12:51:38--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40997929 (39M) [text/plain]
Saving to: '/otus3/pg2600.converter.log'

100%[=====================================================================>] 40,997,929  3.20MB/s   in 11s    

2023-12-03 12:51:49 (3.68 MB/s) - '/otus3/pg2600.converter.log' saved [40997929/40997929]

--2023-12-03 12:51:49--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 40997929 (39M) [text/plain]
Saving to: '/otus4/pg2600.converter.log'

100%[=====================================================================>] 40,997,929  5.12MB/s   in 7.1s   

2023-12-03 12:51:57 (5.48 MB/s) - '/otus4/pg2600.converter.log' saved [40997929/40997929]

```

- Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:

```
[root@zfs ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.6M   330M     21.6M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.2M   313M     39.2M  /otus4

[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                  -
otus2  compressratio         2.22x                  -
otus3  compressratio         3.65x                  -
otus4  compressratio         1.00x                  -

```
> Таким образом, у нас получается, что алгоритм gzip-9 самый эффективный по сжатию.

## Определение настроек пула

- Скачиваем архив в домашний каталог: 

```
[root@zfs ~]# wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download' 
--2023-12-03 12:57:38--  https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
Resolving drive.google.com (drive.google.com)... 173.194.222.194, 2a00:1450:4010:c1e::c2
Connecting to drive.google.com (drive.google.com)|173.194.222.194|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download [following]
--2023-12-03 12:57:39--  https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
Reusing existing connection to drive.google.com:443.
HTTP request sent, awaiting response... 303 See Other
Location: https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/acg9u9245m2ckr47041msnt649826fha/1701608250000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=8eb8bf5e-f4b5-4c1d-8b5b-88bdb7ebf0e2 [following]
Warning: wildcards not supported in HTTP.
--2023-12-03 12:57:44--  https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/acg9u9245m2ckr47041msnt649826fha/1701608250000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=8eb8bf5e-f4b5-4c1d-8b5b-88bdb7ebf0e2
Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 64.233.165.132, 2a00:1450:4010:c08::84
Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|64.233.165.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/x-gzip]
Saving to: 'archive.tar.gz'

100%[=====================================================================>] 7,275,140   7.79MB/s   in 0.9s   

2023-12-03 12:57:46 (7.79 MB/s) - 'archive.tar.gz' saved [7275140/7275140]
```

- Разархивируем его:

```
[root@zfs ~]# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```

- Проверим, возможно ли импортировать данный каталог в пул:

```
[root@zfs ~]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
```
> Данный вывод показывает нам имя пула, тип raid и его состав.

- Сделаем импорт данного пула к нам в ОС:

```
[root@zfs ~]# zpool import -d zpoolexport/ otus

[root@zfs ~]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors
```
> Команда zpool status выдаст нам информацию о составе импортированного пула. Имя пула можно поменять во время импорта: zpool import -d zpoolexport/ otus newotus

- Далее нам нужно определить настройки

```
[root@zfs ~]# zpool get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupditto                     0                              default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.09M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  0%                             -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      8561496395391605048            -
otus  autotrim                       off                            default
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local
```

- C помощью команды grep можно уточнить конкретный параметр, например:

  • Размер: zfs get available otus

```
[root@zfs ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
```
  • Тип: zfs get readonly otus

```
[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
```
  • Значение recordsize: zfs get recordsize otus

```
[root@zfs ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
```
  • Тип сжатия (или параметр отключения): zfs get compression otus

```
[root@zfs ~]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
```
  • Тип контрольной суммы: zfs get checksum otus

```
[root@zfs ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```

## Работа со снапшотом, поиск сообщения от преподавателя

- Скачаем файл, указанный в задании:

```
[root@zfs ~]# wget -O otus_task2.file --no-check-certificate "https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download"
--2023-12-03 13:13:42--  https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
Resolving drive.google.com (drive.google.com)... 173.194.222.194, 2a00:1450:4010:c1e::c2
Connecting to drive.google.com (drive.google.com)|173.194.222.194|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download [following]
--2023-12-03 13:13:42--  https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
Reusing existing connection to drive.google.com:443.
HTTP request sent, awaiting response... 303 See Other
Location: https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/so9o9jqa5fcgke171e7mjqgj38k3kevu/1701609225000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download&uuid=462065b6-2939-4c0b-9336-ecc026e593f1 [following]
Warning: wildcards not supported in HTTP.
--2023-12-03 13:13:46--  https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/so9o9jqa5fcgke171e7mjqgj38k3kevu/1701609225000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download&uuid=462065b6-2939-4c0b-9336-ecc026e593f1
Resolving doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)... 64.233.165.132, 2a00:1450:4010:c0a::84
Connecting to doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)|64.233.165.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5432736 (5.2M) [application/octet-stream]
Saving to: 'otus_task2.file'

100%[===========================================================================>] 5,432,736   6.41MB/s   in 0.8s   

2023-12-03 13:13:48 (6.41 MB/s) - 'otus_task2.file' saved [5432736/5432736]
```

- Восстановим файловую систему из снапшота: zfs receive otus/test@today < otus_task2.file

```
[root@zfs ~]# zfs receive otus/test@today < otus_task2.file
[root@zfs ~]# zfs list
NAME             USED  AVAIL     REFER  MOUNTPOINT
otus            4.96M   347M       25K  /otus
otus/hometask2  1.88M   347M     1.88M  /otus/hometask2
otus/test       2.85M   347M     2.83M  /otus/test
otus1           21.6M   330M     21.6M  /otus1
otus2           17.7M   334M     17.6M  /otus2
otus3           10.8M   341M     10.7M  /otus3
otus4           39.2M   313M     39.2M  /otus4
```

- Далее, ищем в каталоге /otus/test файл с именем “secret_message”:

```
[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
```
- Смотрим содержимое найденного файла:
```
[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome
```
> Текст из файла **secret_message**: <https://github.com/sindresorhus/awesome>

## Vagrantfile для конфигурации сервера (установки и настройки ZFS) с отдельным Bash-скриптом добавленым в Vagrantfile
 
- [Vagrantfile](Vagrantfile)
- [Скрипт для конфигурации сервера (установки и настройки ZFS)](setup_zfs.sh)

- Проверка после развертывания с новым Vagrantfile

```
elena_leb@ubuntunbleb:~/ZFS_DZ$ vagrant ssh
Last login: Sun Dec  3 12:40:48 2023 from 10.0.2.2
[vagrant@zfs ~]$ zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  4.96M                  -
otus  available             347M                   -
otus  referenced            25K                    -
otus  compressratio         1.23x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         25K                    -
otus  usedbychildren        4.94M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.04x                  -
otus  written               25K                    -
otus  logicalused           4.57M                  -
otus  logicalreferenced     13K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               off                    default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
[vagrant@zfs ~]$ zfs list
NAME             USED  AVAIL     REFER  MOUNTPOINT
otus            4.96M   347M       25K  /otus
otus/hometask2  1.88M   347M     1.88M  /otus/hometask2
otus/test       2.85M   347M     2.83M  /otus/test
otus1           21.6M   330M     21.6M  /otus1
otus2           17.7M   334M     17.6M  /otus2
otus3           10.8M   341M     10.7M  /otus3
otus4           39.2M   313M     39.2M  /otus4
[vagrant@zfs ~]$ zfs get all | grep compressratio | grep -v ref
otus             compressratio         1.23x                  -
otus/hometask2   compressratio         1.00x                  -
otus/test        compressratio         1.32x                  -
otus/test@today  compressratio         1.32x                  -
otus1            compressratio         1.81x                  -
otus2            compressratio         2.22x                  -
otus3            compressratio         3.65x                  -
otus4            compressratio         1.00x                  -
vagrant@zfs ~]$ zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors

```


