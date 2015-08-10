Commands to get rocksteady up and running.  Assuming debian

## Dependecies ##
Use your OS equivalent of debian apt-get for following packages.

**`sudo apt-get install sun-java6-jre mysql-server rabbitmq-server`**



## Running from Distribution Build ##

Download newest distribution from http://code.google.com/p/rocksteady/downloads/list

```
unzip rocksteady-*
cd rocksteady-*
#Setup SQL table and create rocksteady user and privilege
sudo mysql < setup.sql
./start.sh 
```

Start.sh will attempt to download Esper binary if it wasn't there and start Rocksteady.

## Running from Source ##

**`svn checkout http://rocksteady.googlecode.com/svn/trunk/ rocksteady-read-only`**

Enter the code directory

**`cd rocksteady-read-only`**

Setup SQL table and create rocksteady user and privilege

**`sudo mysql < setup.sql`**

Install maven ( Java source build helper)

**`sudo apt-get install maven2`**

`*`_The maven version on debian repo might be too old, if that's the case, [download maven](http://maven.apache.org/download.html) tar ball, untar it somewhere, and set a command alias of mvn to the newly downloaded maven binary._

**`alias mvn=$HOME/apache-maven/bin/mvn`**

Run this command to download all rocksteady's Jar dependencies.  Jar is like a tar ball for java to distribute compile binary.

**`mvn package`**

If anything failed to download, create an "issue" here for others to troubleshoot.  This command will also create distribution zip files.

If everything looks good, there will be a new directory called "target" which all the compiled code and library sit in.

**`cd target`**

Run dev mode, useful when testing your customization.

**`../run_dev.sh`**

run\_dev invokes "mvn compile" then starts rocksteady in dev mode with more verbose log.

## Now What? ##
Now that your instance of Rocksteady is running, we want to start to feed it some metrics.  Easiest way to do so is to use message publishing scripts that come with Rocksteady.  See [test metric sending](MetricTestSending.md) for detail.  For full metric sending information, see MetricSending.