# What MHA does on monitoring and failover

Here are basic flows of what MHA([masterha_manager](masterha_manager)) does on master monitoring and failover.

## Verifying replication settings and identifying the current master

  * Identifying the current master by connecting all hosts described in a config file. You do not have to specify which host is the current master. MHA automatically checks replication settings and identify the current master. > (Note: MHA does not build replication environments (i.e. adding/dropping slaves) by itself. MHA monitors master on an existing replication environment)

  * If any slave is dead at this stage, terminating the script for safety reasons(If any slave is dead, MHA can not recover the dead slave, of course).

  * If any necessary scripts is not installed on each MySQL server (MHA Node), MHA aborts at this stage and does not start monitoring.

## Monitoring the master server

  * Waiting until current master server dies
  At this stage, MHA does not monitor slave servers. Stopping/Restarting/Adding/Removing slaves do not have any effect to currently monitoring MHA. When you add/remove slaves, you should update the config file and had better restart MHA.

## Detecting the master server failure

  * If MHA fails to connect the master server three consecutive times, it enters this phase
  * If you set [secondary_check_script](Parameters#secondary_check_script) parameter in a config file, MHA calls the script to double check the master is really dead

> The below steps are also executed by [masterha_master_switch](masterha_master_switch) command. You can use the same arguments with masterha_manager.

## Verifying slave configurations again

  * Reading configuration files again, and connecting to all hosts and verifying the current master dies and all other slaves replicating from the dead master. If any invalid replication configuration is detected (i.e. some slaves replicate from different master), MHA stops with error here. You can change this behavior by setting [ignore_fail](Parameters#ignore_fail) parameter at a configuration file.
  This step is for safety reason. Since it may take very long time (weeks/months) after starting [masterha_manager](masterha_manager), replication configuration might be changed, so double check is generally recommended.

  * Checking last failover status
  If the last failover ends with error, or the last failover finished too recently, MHA stops here and does not start failover. You can change this behavior by ignore_last_failover and wait_on_failover_error arguments at [masterha_manager](masterha_manager) command.

## Shutting down failed master server (optional)

  * If you have defined [master_ip_failover_script](Parameters#master_ip_failover_script) and/or [shutdown_script](Parameters#shutdown_script) in the config file, MHA calls these scripts.
    * Invalidating IP address of the current master
  * Power off the dead master, to avoid split brain

## Recovering a new master
  * Saving binary log events from the crashed master (if possible)
    * If the dead master is reachable via SSH, copying binary logs from the latest slave's end_log_pos (Read_Master_Log_Pos)

  * Determining the new master
    * Based on configuration file settings and current MySQL settings
    * If you have some hosts for candidate master, set [candidate_master=1](Parameters#candidate_master) in the config file.
    * If you have some hosts that never should be a master, set [no_master=1](Parameters#no_master) in the config file.

  * Identifying the latest slave (a slave that has received most recent relay logs)
  * Recovering and promoting the new master
    * Generating differential binary/relay log events to the master
    * Applying the log events to the new master
    * If any error happens here (i.e. duplicate key error), MHA aborts and the below recovery steps including recovering the rest slaves won't take place.

## Activating the new master

  * If you have defined [master_ip_failover_script](Parameters#master_ip_failover_script) in the config file, MHA calls the script
    * You can do anything such as activating IP address of the current master, creating a privileged user, etc

## Recovering the rest slaves

  * Recovering the rest slaves
    * Generating differential binary/relay log events to the slaves in parallel
    * Applying the log events to the slaves in parallel
    * Starting replication from the new master
    * MHA does not abort even if recovery error happens at this stage (i.e. duplicate key error on slave 3). The failed slave won't replicate from the new master, but other successfully recovered slaves can start replication.

## Notifications (optional)

  * If you have defined [report_script](Parameters#report_script) in the config file, MHA calls the script. You can do whatever you like, for example:
    * Sending mails
    * Disabling scheduled backup jobs on the new master
    * Updating internal administration tool status, etc


# What MHA does on online(fast) master switch

> The below steps can be done by [masterha_master_switch](masterha_master_switch) command (with --master_state=alive argument).

## Verifying replication settings and identifying the current master

  * Identifying the current master by connecting all hosts described in a config file (Note: MHA does not build replication environments (i.e. adding/dropping slaves) by itself. MHA monitors master on an existing replication environment)

  * Executing FLUSH TABLES on the current master (optional). This is to minimize the cost(time) of FLUSH TABLES WITH READ LOCK that will be executed later

  * Checking neither MHA master monitoring nor failover is running

  * Checking whether the following conditions are met
    * IO threads on all slaves are running
    * SQL threads on all slaves are running
    * Seconds_Behind_Master on all slaves are less than 2 (seconds) (can be changed by --running_updates_limit=(seconds) argument)
    * On master, none of update query takes more than 2 seconds in the show processlist output

## Identifying the new mater

  * New master host can be set by "--new_master_host" argument. Otherwise new master will be automatically identified from the configuration file.
  * Orig master and new master must have same binary log filtering rules (binlog-do-db and binlog-ignore-db) each other.

## Rejecting writes on the current master

  * If you have defined [master_ip_online_change_script](Parameters#master_ip_online_change_script) parameter in the config file, MHA calls the script.
    * You can gracefully block writes here (i.e. removing a writer user, executing SET GLOBAL read_only=1, etc)
  * Executing FLUSH TABLES WITH READ LOCK on the current master to block all writes (can be skipped by --skip_lock_all_tables argument)

## Waiting for all slaves to catch up replication

  * Using MASTER_LOG_POS() function here

## Granting writes on the new master

  * Executing SHOW MASTER STATUS to identify current binary log file and position of the new master
  * If you have defined [master_ip_online_change_script](Parameters#master_ip_online_change_script) parameter in the config file, MHA calls the script. You can create a writer user, executing SET GLOBAL read_only=0, etc.

## Switching replication on all the rest slaves

  * Executing CHANGE MASTER, START SLAVE in parallel