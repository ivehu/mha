#master_ip_failover_script脚本自动切换mysql master
#杀掉主库（db51）mysql进程，模拟主库发生故障，进行自动failover操作
[root@db51 ~]# pkill -9 mysqld

#查看MHA切换日志，了解整个切换过程（db53）
[root@db53 ~]# cat /var/log/masterha/app1/manager.log
----- Failover Report -----

app1: MySQL Master failover 192.168.5.51(192.168.5.51:9106) to 192.168.5.52(192.168.5.52:9106) succeeded

Master 192.168.5.51(192.168.5.51:9106) is down!

Check MHA Manager logs at db53.blufly.com:/var/log/masterha/app1/manager.log for details.

Started automated(non-interactive) failover.
Invalidated master IP address on 192.168.5.51(192.168.5.51:9106)
The latest slave 192.168.5.52(192.168.5.52:9106) has all relay logs for recovery.
Selected 192.168.5.52(192.168.5.52:9106) as a new master.
192.168.5.52(192.168.5.52:9106): OK: Applying all logs succeeded.
192.168.5.52(192.168.5.52:9106): OK: Activated master IP address.
192.168.5.53(192.168.5.53:9106): This host has the latest relay log events.
Generating relay diff files from the latest slave succeeded.
192.168.5.53(192.168.5.53:9106): OK: Applying all logs succeeded. Slave started, replicating from 192.168.5.52(192.168.5.52:9106)
192.168.5.52(192.168.5.52:9106): Resetting slave info succeeded.
Master failover to 192.168.5.52(192.168.5.52:9106) completed successfully.
#此时192.168.5.52已经是新的master了


#通过master_ip_online_change脚本手动在线切换mysql master
#通过MHA Manger监控，查看集群里面现在谁是master
[root@db53 ~]# masterha_check_status --conf=/etc/masterha/app1.cnf
app1 (pid:26244) is running(0:PING_OK), master:192.168.5.52

#首先，停掉MHA监控：
[root@db53 ~]# /etc/init.d/mha_manager stop

#查看manager status
[root@db53 ~]# masterha_check_status --conf=/etc/masterha/app1.cnf
app1 is stopped(2:NOT_RUNNING).

#其次，进行在线切换操作（模拟在线切换主库操作，原主库192.168.5.52变为slave，192.168.5.51提升为新的主库）
[root@db53 ~]# masterha_master_switch --conf=/etc/masterha/app1.cnf --master_state=alive --new_master_host=192.168.5.51 --new_master_port=9106 --orig_master_is_new_slave --running_updates_limit=10000
Fri Sep 21 12:44:52 2018 - [info] MHA::MasterRotate version 0.56.
Fri Sep 21 12:45:01 2018 - [info]  192.168.5.51: Resetting slave info succeeded.
Fri Sep 21 12:45:01 2018 - [info] Switching master to 192.168.5.51(192.168.5.51:9106) completed successfully.

#通过MHA Manger监控，验证一下集群里面的mysql master是不是已经切换到192.168.5.51
[root@db53 ~]# /etc/init.d/mha_manager start 
[root@db53 ~]# masterha_check_status --conf=/etc/masterha/app1.cnf
app1 (pid:19644) is running(0:PING_OK), master:192.168.5.51
