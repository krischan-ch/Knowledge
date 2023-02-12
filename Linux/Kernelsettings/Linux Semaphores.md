# What are Semaphores?
(what are they used for and example exa301s90)

## description & usage

(taken from https://datacadamia.com/os/linux/semaphore)

On Linux, A semaphore is a IPC (InterProcessCommunication) object that is used to control utilization of a particular process.

Semaphores are a **shareable** resource that take on a non-negative integer value. They are manipulated by the P (wait) and V (signal) functions, which decrement and increment the semaphore, respectively. When a process needs a resource, a “wait” is issued and the semaphore is decremented. When the semaphore contains a value of zero, the resources are not available and the calling process spins or blocks (as appropriate) until resources are available. When a process releases a resource controlled by a semaphore, it increments the semaphore and the waiting processes are notified.

## current setting

Get the current setting from /proc/sys/kernel/sem file:

`root@exa301s90:~/ [TTX81] cat /proc/sys/kernel/sem`
`1024    80000   1024    256`

The output lists, in order, the values for the **semmsl**, **semmns**, **semopm**, and **semmni** parameters.

## maximum values

| name | description | maximum (MoS Note [1670658.1](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=166460034084699&id=1670658.1)) |
| :--- | :--- | :-- |
| semmsl | maximum number of semaphores in a semphore set | 65'536 |
| semmns | maximum number of semaphores in the system | 2'147'483'647 |
| semopm | maximum number of operations per semop(P) call | OS.dependent |
|  semmni | maximum number of semaphore sets in system | 32'768 |

## calculation

as of My Oracle Support Note [15654.1](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=153654195786773&id=15654.1)

1. get the values for parameter **processes** of all databases (dbinfo.pl -p processes) 
2. sum the values from step 1 (exa301s90:66'850) and add 10% for "any other system requirements" (exa301s90:73'535) = number of semaphores needed to start all databases ("AT LEAST" value for semmns)
4. Semaphores are allocated by Unix in 'sets' of up to semmsl 
            semaphores per set. You can have a MAXIMUM of semmni sets on the
            system at any one time. semmsl is an arbitrary figure which is 
            best set to a round figure no smaller that the smallest 'processes'
            figure for any database on the system. This is not a requirement 
            though.
            (exa301s90: semaphore sets needed = processes / semmsl (=500 -> minimum processes Parameter))
            (exa301s90: sum of result for each instance: 139 + 10% = 153)


result: sem = 500&emsp;73535&emsp;1000&emsp;153

## conclusion

The values currently set on exa301s90 are all higher than the calculated ones. 
The first one (semmsl) is much higher than the calculated one. So this ensures that there are quite enough semaphores in each set.

The second one (semmns) is similar to the currently set one. So this need to be increased vastly if we need to create more databases.

The third one (semopm) is slightly below the currently set one. The maximum recommended value as of MoS note [1670658.1](https://support.oracle.com/epmos/faces/DocContentDisplay?_afrLoop=167762430355847&id=1670658.1). As the currently set one was set by Oracle install routine I'd assume that this is fine.

The fourth one (semmni) is much below the currently set one so that one is fine as well.


## links

Semaphore Parameters

  [https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cwlin/tuning-semaphore-parameters.html#GUID-467E41B6-2B11-426E-9447-A4D8BF8E26C6](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cwlin/tuning-semaphore-parameters.html#GUID-467E41B6-2B11-426E-9447-A4D8BF8E26C6)

Semaphore Settings Example

  [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/tuning_and_optimizing_red_hat_enterprise_linux_for_oracle_9i_and_10g_databases/sect-oracle_9i_and_10g_tuning_guide-setting_semaphores-an_example_of_semaphore_settings](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/tuning_and_optimizing_red_hat_enterprise_linux_for_oracle_9i_and_10g_databases/sect-oracle_9i_and_10g_tuning_guide-setting_semaphores-an_example_of_semaphore_settings)

Which DB belongs to which DB

[https://db-blog.web.cern.ch/blog/szymon-skorupinski/2015-04-which-shared-memory-segments-belong-my-database-instance](https://db-blog.web.cern.ch/blog/szymon-skorupinski/2015-04-which-shared-memory-segments-belong-my-database-instance)

Installation of oracle database kernel parameters under Linux

[https://programmer.group/installation-of-oracle-database-kernel-parameters-under-linux.html](https://programmer.group/installation-of-oracle-database-kernel-parameters-under-linux.html)


# Tags:

[[HowTo]] - [[Linux]] - [[KernelSettings]]