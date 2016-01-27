# masterha_manager: Command to run MHA Manager

MHA Manager can be started by executing masterha_manager command.

    # masterha_manager --conf=/etc/conf/masterha/app1.cnf

masterha_manager takes below arguments.

## Common arguments

* --conf=(config file path)

Application and local scope configuration file. This argument is mandatory.

* --global_conf=(global config file path)

Global scope configuration file. Default is /etc/masterha_default.cnf

* --manager_workdir, --workdir

Same as [manager_workdir](Parameters#manager_workdir) parameter.

* --manager_log, --log_output

Same as [manager_log](Parameters#manager_log) parameter.

## Monitoring specific arguments ##

* --wait_on_monitor_error=(seconds)

If an error happens during monitoring, masterha_manager sleeps wait_on_monitor_error seconds and exits. The default is 0 seconds(not waiting). This functionality is mainly introduced for doing automated master monitoring and failover from daemon scripts. It is pretty reasonable for waiting for some time on errors before restarting monitoring again.

* --ignore_fail_on_start

By default, master monitoring (not failover) process stops if one or more slaves are down, regardless of "ignore_fail" parameter setting. By setting --ignore_fail_on_start, master monitoring does not stop if ignore_fail marked slaves are down.

## Failover specific arguments ##

* --last_failover_minute=(minutes)

If the previous failover was done too recently (8 hours by default), MHA Manager does not do failover because it is highly likely that problems can not be solved by just doing failover. The purpose of this parameter is to avoid ping-pong failover problems. You can change time criteria by changing this parameter. The default is 480 (8 hours).

If --ignore_last_failover is set, this step is ignored.

* --ignore_last_failover

If the previous failover failed, MHA does not start failover because the problem might happen again. The normal step to start failover is manually remove failover error file which is created under (manager_workdir)/(app_name).failover.error .

By setting --ignore_last_failover, MHA continues failover regardless of the last failover status.

* --wait_on_failover_error=(seconds)

If an error happens during failover, MHA Manager sleeps wait_on_failover_error seconds and exits. The default is 0 seconds (not waiting). This functionality is mainly introduced for doing automated master monitoring and failover from daemon scripts. It is pretty reasonable for waiting for some time on errors before restarting monitoring again.

* --remove_dead_master_conf

When this option is set, if failover finishes successfully, MHA Manager automatically removes the section of the dead master from the configuration file. For example, if the dead master's hostname is host1 and it belongs to the section of server1, the entire part of the server1 will be removed from the configuration file.
By default, the configuration file is not modified at all. After MHA finishes failover, the section of the dead master still exists. If you start masterha_manager immediately (this includes automatic restart from any daemon program), masterha_manager stops with an error that "there is a dead slave" (previous dead master). You might want to change this behavior especially if you want to continuously monitor and failover MySQL master automatically. In such cases, --remove_dead_master_conf argument is helpful.