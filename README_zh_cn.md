# xtrabackup
This image uses martinhelmich/xtrabackup as a base and tries to provide an
entrypoint to support full backup and restore procedures from within rancher

## Use case
The image is built with the following use case in mind:

- 运行的Mysql通过mysql.conf的配置，将data与log分开保存，此工具挂载mysql.conf配置来定位data和log的位置，来进行备份和恢复。
- 备份将被保存在/target目录下，该目录可以挂载外部存储来保存。
- 恢复也通过mysql.conf配置进行。能够将data和log一起恢复。并加入并行10个线程的参数加快备份恢复速度。
- 去掉了rancher等环境变量和函数，没用的地方或去掉或改动了。适用于在k8s环境下运行备份恢复mysql数据库。传入相应环境变量即可，可以控制备份名称。

## Usage
The container knows the following commands. Starting the container without a
command will prompt the container to print its usage information.

- backup
  Use xtrabackup to create a backup to /target/YY-mm-dd-HH\_ii
  - Set environment variable `BACKUP_MODE=FULL` to create a backup to /target/full-YY-mm-dd-HH\_ii and do not prepare it
  - Set environment variable `BACKUP_MODE=INCREMENTAL` to create an incremental backup based on the latest full backup found to
    /target/full-FULL_BACKUP_TIMESTAMP-inc-YY-mm-dd-HH\_ii
- restore BACKUP
  Attempts to restore the given BACKUP from /target/BACKUP
  - If the regex `^full-.*-inc-.*` matches the backup name it is treated as incremental backup and the full backup
    included in its name is restored before the incremental one.
- clear
  Clears the backup directory in preparation for restoring a backup. This is
  separate from restore to allow this command to run on all hosts of a galera
  cluster
- run COMMAND
  Run the given command within the container
- cleanup
  Remove the full backup 4 weeks older than the last one
- remove BACKUP
  Remove the given full backup, its incremental childs and their respective restore services
- help
  Print Usage

### backup

| Environment variable | xtrabackup parameter | defaults to |
| -------------------- | ------------------- | ----------- |
| MYSQL\_CONF\_FILE | --defaults-file $PARAM | - |
| MYSQL\_HOST | --host $PARAM | target |
| MYSQL\_PORT | --port $PARAM | 3306 |
| MYSQL\_USER | --user $PARAM | - |
| MYSQL\_PASSWORD | --password $PARAM | - |

Current behavious is to prepare the backup fully after creating it. This will
probably change in the future as we start implementing incremental backups and
preparing on utility servers.

### restore
- 恢复分2步，先prepare，然后copy-back。原脚本为了备份可用于增量备份，采用复制一份备份的方式来做。数据库太大的话，就直接在备份上进行prepare了。
- prepare后，将原data和log移动备份，然后copy-back，会恢复data和log
- 任务最后，查看xtraback_info文件，是为了获取GTID等信息，直接查容器的最后20行日志，即可获取。

### clear
Clean the mysql database data via `rm -Rf /var/lib/mysql/*`.  
This is intended to be used with `run on all hosts` and scheduling rule `must
have service DBSTACK/NEWDBSERVICE` to prepare the new database cluster to receive
the backup

### run COMMAND
Runs the given command as command line within the containers. This is intended
to debug the container.

### help
Prints usage.  
Currently only prints the full overview.

help [COMMAND] might be included in the future if the need arises

## Example
- Step 1: Create backup from existing data which is not within  

	```sh
	docker volume create --driver=convoy --name=backup
	docker run -it --volumes-from DB_DATA_VOLUME_CONTAINER -v backup:/target --link DB_SERVER_CONTAINER:target -e MYSQL\_PORT=3306 -e MYSQL\_USER=root -e MYSQL_PASSWORD='PASSWORD' ipunktbs/xtrabackup
	```

- Step 2: Create new PXC cluster from Rancher Catalog
- Step 3: Upgrade PXC service to match your scheduling needs
- Step 4: Stop PXC service
- Step 5: Create ipunktbs/xtrabackup service with
  - Command: clear
  - Scheduling: All hosts, must have service DBSTACK/DBSERVICE
  - Volumes: PXC\_NAMED\_MYSQL\_VOLUME:/var/lib/mysql
- Step 6: Delete the clear service after all instances have successfully finished
- Step 7: Create ipunktbs/xtrabackup service with
  - Command: restore
  - Scheduling: Scale 1, must have service DBSTACK/DBSERVICE
  - Volumes: PXC\_NAMED\_MYSQL\_VOLUME:/var/lib/mysql, backup:/target
  - Note: You might have to do this using rancher-compose until per-volume driver is available in rancher, or create the `backup` volume with a driver on all hosts
- Step 8: Start the pxc server on the server where the `restore` container ran
  and run the command `SET GLOBAL wsrep_provider_options="pc.bootstrap=1";` to
  make it the new master.
- Step 9: Start the other pxc servers and wait for the state sync
