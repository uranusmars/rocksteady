Rocksteady is a java application that reads metrics from RabbitMQ, parse them and turn them into events so Esper(CEP) can query against those metric and react to events match by the query.

## Overview ##
Rocksteady is an effort to utilize complex event process engine to analyze user defined metric.  End goal is to derive root cause conclusion based on metric driven events.  Rocksteady is only the metric analysis part of the whole picture, but we also present a solution including metric convention, metric sending, load balancing, and graphing that work well for us.

## Audience ##

Ops not scare of getting their hand dirty with a bit of coding and devs who want to monitor their app with metric instead of gut feeling.

## Background ##

Too often after collecting many metric and creating many pretty graphs, one realizes graph serves only as a good postmortem analytic aid.  Staring at dozen of graph on a TV wall isn't monitoring, it's a waste of time.  We want to look for problems as metric being collected(real time).  We need a way to express our troubleshooting knowledge in code so similar problems can be detected and correlated.  We need to process LOTS of metrics together in order to make sense of our own world.

This project is written by an ops guy who want to learn Java and solve his metric problem.  There  will probably be things that a real programmar snorts at, criticism is welcome.  The documents will probably cover some obvious things in detail because I don't want my fellow ops to be as confused as I was when first enter the wonderful world of Java programming.

## Design ##

This map shows where rocksteady sits in the environment.

![http://farm5.static.flickr.com/4140/4924106126_528aef01e0.jpg](http://farm5.static.flickr.com/4140/4924106126_528aef01e0.jpg)

  1. Metrics sent from hosts into rabbitmq, which would look something like **1min.juicer.system.cpu.prct\_idle.dc1.pi101 82 1283192317**.  Read detail at [metric format](MetricFormat.md)
  1. Rocksteady subscribe to metric exchange on rabbitmq.  It can also publish its own metric into rabbitmq.
  1. Graphite subscribe to metric exchange on rabbitmq.
  1. Rocksteady can request some historic metric from graphite.
  1. Rocksteady process the metric and alert nagios.
  1. Rocksteady capture some data, either raw or composite, into db.
  1. Graphite data used in dashboard.
  1. Rocksteady captured data used in dashboard.

Take a look at [Design Background](DesignBackground.md) to see why we did what we did.


## Requirements ##

Rocksteady is an application written in Java using Spring framework, build with Maven, and should run wherever you can install Java.

Rocksteady needs:
  * Java 1.5+
  * [Maven2](http://maven.apache.org/) (It's like apt-get + ./configure + make for Java)
  * [Rabbitmq](http://www.rabbitmq.com/) + [shovel plugin(optional)](http://www.lshift.net/blog/2010/02/01/rabbitmq-shovel-message-relocation-equipment)
  * Mysql (Track historic data)

Optional:
  * [Graphite](http://graphite.wikidot.com/) (Orbitz open sourced graphing backend)
  * [GLP](http://www.mail-archive.com/graphite-dev@lists.launchpad.net/msg00411.html), graphite local proxy.  Useful to collect metric, install on each server to collect metric from localhost and forward to configured rabbitmq exchange.

## Installation ##
See [Installation Notes](InstallNotes.md) for detail on installation.

For binary installation:
Download newest distribution from http://code.google.com/p/rocksteady/downloads/list .

```
unzip rocksteady-*
cd rocksteady-*
./start.sh 
```


## Quickstart ##
  1. Rocksteady depends on metric being certain format, specifically [graphite metric format](MetricFormat.md).
  1. If the default metric convention suits your environment, start to [send some metric](MetricTestSending.md) to rabbitMQ for testing.  Leave it running while you are testing Rocksteady.
  1. Modify [configs](Configuration.md) to look for new metric if you made any. Default installation has one query that makes minutely aggregation from test metrics.  (Remember to restart Rocksteady if you make change.)
  1. Play with [email sending, data capturing, or nagios alerting](EplListener.md) when query finds something.  (Remember to restart Rocksteady if you make change.)

## What's Next ##
The important thing we learn that enable us to use Rocksteady is metrics.  We need to make exposing metric as easy as possible.  We made that happen by using a simple metric string convention, a local metric listening port, Rabbitmq and Graphite.  With sending metric and seeing its graph being so easy, developers are willing to create them.  After we start to have metric, the fun begins.

Now that you have a glimpse of CEP power on metric, there really is no boundary what we can do with metrics.  One thing for sure is that you NEED to customize Rocksteady.  Like database where everyone has different tables, different field names, we will have different metrics for different people.

Here is one of the way we use Rocksteady.  Consider request per second(rps) has a time characteristic where it goes up and down according to time of the day.  And slew of other metrics such as cpu and network move accordingly to rps.  We can then use prediction algorithm such as Holt Winter to predict a confidence band for next arriving value and record the event whenever metrics cross the band consecutively in certain number of times.  This is what we call auto threshold establishment.  Now, if we have a SLA we care dearly such as response time, we can set a hard threshold, say 250ms.  When response time gets slower than 250ms, we check to see if rps, cpu or network cross their thresholds.  So now instead of just knowing latency problem, we can quickly pin point potential causes.

One trick we did was exposed version of application code as metric.  Every time that metric changes, it means new code was deployed and application was restarted.  We had Rocksteady tracks these events and persist to database so when important metric such as response time slows down, we always include time of last deployment.  Some problem such as memory leak could take hours to days to surface, so having quick peak to last deploy time is handy.

Sounds too rosy to be true?  Here's the hard work.  In order to build correlation query, we need to understand which metrics are correlated, and this is very application specific and requires time spent on looking at metric for a while.  Previously learned knowledge such as "if number of records goes over 5 million, garbage collection will kill the performance" should be put into query(provided this metric is available) so when latency goes up, these things are checked automatically so they can be ruled out.

You can see from EPL examples, there are queries just recording threshold breaching events, those event themselves don't trigger alert, but are used by other query for cause finding.  Here's our workflow.  We often start with an important metric such as latency, set a threshold, and capture bunch of potential related metric to database whenever latency cross threshold.  We build a simple dashboard talk to DB to inspect latency event along with other captured metric to see if they are related.  Because these metrics are also recorded by Graphite, we can draw a small 30-minutes time frame spark line graphe from Graphite next to each metric helping to see correlation.  Once we identify related metric, we set their threshold so they become part of the correlation chain when latency tanks.  Of course, if we were able to establish threshold for components, we can easily have Rocksteady issue preventive action such as spinning up new instance, throttle back traffic or simply fire an alert before metrics reach meltdown threshold.

Project like [esperr](http://illposed.net/esperr.html) putting R(statistical computing) behind Esper can be very interesting, but you probably need some good math folks.  I am sure there are more interesting stuff people can come up with, I hope this project stirs up people's interest with metric and CEP.
