Rocksteady started with [Spring Roo](http://www.springsource.org/roo) to create application skeleton, so a few configuration were created automatically.

**In distribution, config files are in classes/META-INF.**

**In source, config files are in [src/main/resources/META-INF](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/)**

```
malin@kaohsiung:/tmp/rocksteady-1.0.0/classes/META-INF$ ls -lR
.:
total 28
-rw-r----- 1 malin eng  845 Sep  3 14:15 aop-ajc.xml
-rw-r----- 1 malin eng  233 Sep  3 14:15 database.properties
-rw-r----- 1 malin eng  874 Sep  3 14:15 log4j.properties
-rw-r----- 1 malin eng  569 Sep  3 14:15 log4j.properties.production
-rw-r----- 1 malin eng  805 Sep  3 14:15 persistence.xml
-rw-r----- 1 malin eng 1361 Sep  3 14:15 rocksteady.properties
drwxr-x--- 2 malin eng 4096 Sep  3 14:17 spring

./spring:
total 32
-rw-r----- 1 malin eng  5220 Sep  3 14:15 applicationContext.xml
-rw-r----- 1 malin eng  1411 Sep  3 14:15 email.xml
-rw-r----- 1 malin eng  1893 Sep  3 14:15 epl.xml
-rw-r----- 1 malin eng 12056 Sep  3 14:15 eplStatements.xml
-rw-r----- 1 malin eng  2372 Sep  3 14:15 timerTasks.xml
```

**`*.properties` are files with bunch of key=value lines in it.  The value can then be used in configuration XML by referring to key as ${key}.  So for any ${key} refer in XML, just grep that "key" from .properties to find the value.**

|aop-ajc.xml,persistence.xml|Spring Roo auto generated configs, no touchy unless going through Roo.|
|:--------------------------|:---------------------------------------------------------------------|
|[database.properties](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/spring/database.properties)|Database connection setting.                                          |
|log4j.properties{.production}|Logging setting.  Production version is less verbose.                 |
|[rocksteady.properties](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/rocksteady.properties)|App setting, rabbitmq setting, email server, etc.                     |
|[applicationContext.xml](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/spring/applicationContext.xml)|Spring framework configuration.  It tells Spring what to do, including bean initiation, service start/shutdow, pass keys from property files to Java classes.|
|email.xml                  |Email setting and basic template, used in email notification.         |
|[epl.xml](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/spring/epl.xml)|Contains list of query from eplStatements.xml to run against metric we're collecting.  It can also contain epl statements.|
|[eplStatements.xml](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/spring/eplStatements.xml)|Contains actual queries written in Esper Processing Language(similar to SQL).|
|timerTasks.xml             |Contains some timer tasks such as old data purge.                     |