## Introduction ##

Rocksteady was originally design to analyze the same metric string that was meant for [Graphite](http://graphite.wikidot.com/).  The end result turns out to be a solution so easy to work with, we stick with it.  If this scheme doesn't work for you, Rocksteady needs to be modified to understand the new scheme.  I would suggest to give this a try.  Because it's just a simple one liner string to create metric, it was easy to ask others to adopt.


## Format ##

A metric strings takes the following form:
```
retention.app_name.component+.colo.hostname value unix_timestamp
```

|retention|Graphite use it to determine data storing interval.  We use 10sec,1min,5min,15min.  It's arbitrary, as long as Graphite can regex match it.|
|:--------|:------------------------------------------------------------------------------------------------------------------------------------------|
|app\_name|Application Name                                                                                                                           |
|component+|Any number of level of component.  For ex: cpu.user or iostate.tps.device.sda                                                              |
|colo     |Colocation name                                                                                                                            |
|hostname |Short hostname.  Combining with colo, identify exact server sending the metric                                                             |
|value    |Value for the metric.  Float or integer                                                                                                    |
|unix\_timestampe|Epoch timestamp.  Identify time of metric value                                                                                            |

For example, we script around unix sar command to gather system metrics and for cpu they look like the following:

```
1min.juicer.system.cpu.prct_idle.dc1.pi101 82 1283192317
1min.juicer.system.cpu.prct_nice.dc1.pi101 1 1283192317
1min.juicer.system.cpu.prct_iowait.dc1.pi101 5 1283192317
1min.juicer.system.cpu.prct_system.dc1.pi101 7 1283192317
1min.juicer.system.cpu.prct_user.dc1.pi101 5 1283192317
```

Each line is send to MQ as a message.  Graphite picks them up and store in whisper(RRD-like) files for later graphing and query.  Rocksteady picks them up and continuously query against them.