# Unofficial Zabbix template for pgBackRest
This Zabbix template provide simple and probably "good enough" solution for monitoring pgBackRest backups and WAL archives. 

## Features: 

 - Collect data with plain pgBackRest info command (no need for custom scripts, etc.)
 - Build-in stanza autodiscovery
 - Collect almost all metrics exposed by pgBackRest info command (can be extended if needed)
 - All items ate dependent items. This means less trafic and simple setup 
 - Contains predefined triggers for basic issue detection
 - Various aspect can be configured with user macros
 - Stores only significant values (unchanged values are discarded)

## Known issues and limitations

 - It is primarily designed to monitor dedicated pgBackRest repository hosts. It should also work on shared hosts (pgBackRest and Zabbix agent on the same host as Postgres), but I newer tested that. 
 - It does not support multiple pgBackRest configs on the same host. This is due to fact that Zabbix template can be linked to specific host only once. Of course, you can use some Zabbix-fu to achieve that, but that is outside of "simple" scope of this template.    
 - Per stanza repositories are not supported. This is due to this: https://github.com/pgbackrest/pgbackrest/issues/1628 

## Prerequisites

### Zabbix
In order to use this template, you will need properly configured and working Zabbix Agent on the host with pgBackRest archive and of course Zabbix Server to collect metrics. 

This template should work on Zabbix 5 and newer (tested against 5.4)

### pgBackRest

Should work on all pgBackRest 2.x versions (tested against 2.37) 

## Setup
### Zabbix agent
First, you have to create **pgbackrest.info** Zabbix agent user parameter on the machine with your pgBackRest repository. This parameter should contain command which returns pgBackRest info in machine readable JSON format (`--out=json`). Something like this:

    UserParameter=pgbackrest.info,/usr/bin/sudo -u pgbackrest /usr/bin/pgbackrest --output=json info

You can use whatever command you want, but it must return JSON in the same format as pgbackrest info command.

It is also good idea to put user parameter into standalone .conf file (eg. `pgbackrest.conf`) inside agent configuration ".d" folder (eg. `/etc/zabbix/zabbix_agentd.d`). That way your custom parameters survive during package updates. 

For simplicity, you can grab `pgbackrest.conf` file from this repo and place it accordingly. Then you should be good to go.

#### Permissions
pgBackRest commands should be executed under the user that owns pgBackRest repository. Zabbix agent commands runs under the same user as Zabbix agent daemon. Most likely, these users are not the same. If they are, you can proceed to the next step.  If they are not, you have to switch users before executing pgBackRest commands with Zabbix agent. You can do that with `sudo` subsystem by simply prefixing your command with `sudo -u pgbackrest` But, before you can do this, it is necessary to add following lines to `/etc/sudoers` file:     

    zabbix ALL=(pgbackrest) NOPASSWD: /usr/bin/pgbackrest
    Defaults!/usr/bin/pgbackrest !requiretty

This allow user `zabbix` to execute program `/usr/bin/pgbackrest` as user `pgbackrest` without password. It also instruct sudo that program  `/usr/bin/pgbackrest` does not need terminal to be executed.

Now you can test it by executing following command as zabbix user:

    $ sudo -u pgbackrest pgbackrest version

And of course, you have to change usernames mentioned above to match you real usernames. 

It is also good idea to check SELinux if everything is OK. If not, act acordingly ;)

### Zabbix server
First, you have to import yaml templete from this repository (`template.yaml`) to zabbix server. For that you can follow these [instructions](https://www.zabbix.com/documentation/current/en/manual/xml_export_import/templates#importing).

After import, you can use template on hosts where **pgbackrest.info** user parameter is present. 

#### Macros
You can customise template behavior on each host with these macros:

 - **ARCHIVE_EXPIRE_THRESHOLD** - Threshold for old WALs in archive
 - **ARCHIVE_STALE_THRESHOLD** - Max time without new WALs in archive 
 - **LAST_BACKUP_THRESHOLD** - Last backup maximum age (no matter of type)
 - **LAST_DIFF_BACKUP_THRESHOLD** - Last differential backup maximum age
 - **LAST_FULL_BACKUP_THRESHOLD** - Last full backup maximum age
 - **LAST_INCR_BACKUP_THRESHOLD** - Last incremental backup maximum age
 - **POLLING_INTERVAL** - Agent info polling interval
 - **RETENTION_DERIVED_JSON** - Retention period for derived JSON objects
 - **RETENTION_INFO_JSON** - Retention period for raw info JSON object
 - **RETENTION_ITEMS** - Retention period for basic items
 - **RETENTION_TREND_ITEMS** - Retention period for trends

## How it works
It is pretty simple. Zabbix server polls pgBackRest info json object from Zabbix agent in regular intervals. After each poll, stanza discovery rule is aplied to retrieved data. Then for aeach discovered stanza, coresponding items are captured and stored acording to user macros settings. 




