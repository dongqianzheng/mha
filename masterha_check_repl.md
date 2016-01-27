## Checking MySQL Replication Health by masterha_check_repl

Here is an example.

    manager_host$ masterha_check_repl --conf=/etc/app1.cnf
    ...
    MySQL Replication Health is OK.

masterha_check_repl connects all MySQL servers defined at the config file, then checking replication works correctly or not.
This command does not conflict with masterha_manager. You can run this command while masterha_manager is working (monitoring the current master). This command is useful if you want to check replication settings regularly. After masterha_manager monitors MySQL master, it only checks master server's availability. By running masterha_check_repl regularly and send alerts if it gets errors (i.e. SQL thread stops), you will be able to quickly fix replication problems.

masterha_check_repl returns errors when detecting any replication failure. Replication failures include the followings.
  
* All replication failures that MHA can't monitor/failover (replication filtering rule is different, SQL thread is stopped with errors, if you have two or more masters, etc)
* IO thread is stopped
* SQL thread is stopped normally
* Replication delays more than N seconds


masterha_check_repl takes below arguments, in addition to --conf.

* --seconds_behind_master=(seconds)

  masterha_check_repl checks Seconds_Behind_Master from all slaves and return errors if it exceeds threshold. By default, it's 30 (seconds).

* --skip_check_ssh

  By default, masterha_check_repl executes as the same check scripts as masterha_check_ssh. If you are sure that SSH connection checks are not needed, use this option.