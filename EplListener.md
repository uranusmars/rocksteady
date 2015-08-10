# Introduction #

Listeners in a statement bean is used to play with data an EPL query capture.  This is where most of us will do customization to Rocksteady.  Each of us probably have different need for the data that was captured by EPL, for instance, for some data, it saves them to DB for later analysis, for some, it emails the data.

Sometimes, there will be statement bean without any listener, chances are, they are just collecting data to be used by other statements.



# Details #

As mention before, listener is where most people will do their customization.   "SELECT" in EPL or SQL declare output fields.  Since POJO is what gets fed into Esper, POJO properties are fields available for EPL's "SELECT" as well as aggregation function such as "avg()" or "max()".  We need to make sure listeners are using available fields, otherwise it will cause run time error.

There are couple listener Rocksteady comes with, look for them in [this source location](http://rocksteady.googlecode.com/svn/trunk/src/main/java/com/admob/rocksteady/reactor/).

If you intend to roll out your own listener, refer to "Running from Source" in InstallNotes.  It has instruction for compile and packaging.  Easy, just single maven command.

We'll cover couple listener that's used regularly.
  1. Alerting
  1. Nagios
  1. Graphite

I am hoping by covering two of listener, user can see how easy it is to create your own listener.


## Alerting ##

Alerting reactor is an all purpose reactor(that means messy code).  It is used to record data to db or simply write a log line.  Behavior is determined by value supplied to a property  named "type" in each instance of Alerting bean.  Let's use a statement bean as example:

```
  <bean class="com.admob.rocksteady.router.cep.StatementBean" id="requestTimeOut">
  <constructor-arg value="SELECT irstream *
                         FROM Metric(app='admixer')                         
                         WHERE ((name = 'adworker.timeout_rate' AND value > 5) OR
                         (name = 'services.profiling.timeouts' AND value > 5) OR
                         (name = 'adworker.targeting.timeouts.rate' AND value > 5) )                                     
                         "/>
    <property name="listeners">
     <list>
       <bean class="com.admob.rocksteady.reactor.Alerting">
           <property name="type" value="latency_single"/>
       </bean>
     </list>
    </property>
  </bean>
```

For each event that fits the query, POJO is sent to **Alerting** bean with property named "type" set to "latency\_single".

In [Alerting](http://rocksteady.googlecode.com/svn/trunk/src/main/java/com/admob/rocksteady/reactor/Alerting.java) class, the method **update** is called automatically, it looks like

```
    public void update(EventBean[] newEvents, EventBean[] oldEvents) {
```

Which Esper will feed the method with new POJO that was captured and old POJO that was evicted out of data window.

This is the for loop inside the method to go through the events
```
    for (EventBean newEvent : newEvents) {
      i++;
      try {
        String msg = "";
        String _subject = "";
        // This is how string comparison is done in JAVA, not ==
        if (type.equals("log")) {
          String name = newEvent.get("name").toString();
          // String value = newEvent.get("value").toString();
          String error = newEvent.get("error").toString();
          String count = newEvent.get("count").toString();

          msg = name + " - " + error + " - " + count + " errors.";

        } else if (type.equals("latency_single")) {
          String hostname = newEvent.get("hostname").toString();
          String name = newEvent.get("name").toString();
          String value = newEvent.get("value").toString();
          String colo = newEvent.get("colo").toString();
          String app = newEvent.get("app").toString();

          String graphiteTs = newEvent.get("timestamp").toString();


          msg = colo + " - " + hostname + " - " + app + " - " + name + " - " + value;

          Threshold threshold = new Threshold();
          threshold.setName(name);
          threshold.setHostname(hostname);
          threshold.setColo(colo);
          threshold.setApp(app);
          threshold.setGraphiteTs(graphiteTs);
          threshold.setGraphiteValue(value);
          threshold.setCreateOn(sqlDate);
          threshold.persist();

          msg = msg + "\n" + "http://" + rsweb + "/rsweb/threshold.php?id="
              + String.valueOf(threshold.getId());
          _subject = this.subject + " - " + colo + " - " + hostname;
      }
```
So fo latency\_single, it expects the event to have hostname,name,value,colo, app and timestamp.  It builds a message based on those info, it then save the data to database, and append to msg with a web link to an app "rsweb" that's used for manual event correlation.  The **msg** can then print to log or email out if we supplied a recipients property like this:
```
       <bean class="com.admob.rocksteady.reactor.Alerting">
           <property name="type" value="averagedThreshold"/>
           <property name="recipients">
             <list>
               <value>email@myteam.com</value>
             </list>
           </property>
       </bean>
```

## Nagios ##
Initiate it same way as other listener.  Need to set two property, "level" and "service".  Level is the level of criticality, 1=warning, 2=critical, 3=unknown.  Service is the service name to trigger.

```
       <bean class="com.admob.rocksteady.reactor.Nagios">
           <property name="service" value="request_timeout"/>
           <property name="level" value="2"/>
       </bean>
```

Here's the update method for [Nagios](http://rocksteady.googlecode.com/svn/trunk/src/main/java/com/admob/rocksteady/reactor/Nagios.java) class, events need to have name, value, and colo.  It will send the alert with hostname set to localhost, then message set to "msg" variable.
```
  public void update(EventBean[] newEvents, EventBean[] oldEvents) {
    if (newEvents == null) {
      return;
    }
    for (EventBean newEvent : newEvents) {
      try {
        String name = newEvent.get("name").toString();
        String value = newEvent.get("value").toString();
        String colo = newEvent.get("colo").toString();
        String msg;
        logger.info(
            " event triggered - type " + type + " - " + colo + " - " + name + " - " + value);

        msg = " event triggered - type " + type + " - " + colo + " - " + name + " - " + value;

        // This builds the message to be sent to nagios.
        MessagePayload payload = new MessagePayloadBuilder()
            .withHostname("localhost")
            .withLevel(level)
            .withServiceName(service)
            .withMessage(msg)
            .create();

        nagiosSender.send(payload);

      } catch (NagiosException e) {
        logger.error("Nagios sending exception: " + e.getMessage());
      } catch (Exception e) {
        logger.error("Problem with event: " + newEvent.toString());
      }

    }
  }
```

Nagios server configuration is declared here
```
  @Autowired
  private NagiosPassiveCheckSender nagiosSender;
```

From applicationContext.xml, one can find nagiosSender bean. Value for variable like ${nagiosServer} can be find in rocksteady.properties file.

```
  <!-- This is our nagios sender -->
  <bean class="com.googlecode.jsendnsca.core.NagiosPassiveCheckSender" id="nagiosSender">
    <constructor-arg>
		  <bean class="com.googlecode.jsendnsca.core.NagiosSettings" id="nagioSettings">
		    <property name="nagiosHost" value="${nagiosServer}"/>
		    <property name="password" value="${nagiosNscaPassword}"/>
		    <property name="port" value="${nagiosNscaPort}"/>
		  </bean>     
    </constructor-arg>
  </bean>
```

## Graphite ##
Following listener will send captured metric to configured graphite host.

```
    <property name="listeners">
     <list>
       <bean class="com.admob.rocksteady.reactor.Graphite">
           <property name="suffix" value="averaged"/>
       </bean>    
     </list>
    </property>
```

Here's what Graphite class expects to have in the class, and append supplied "suffix" property to original metric name.
```
  public void update(EventBean[] newEvents, EventBean[] oldEvents) {

    if (newEvents == null) {
      return;
    }
    for (EventBean newEvent : newEvents) {
      try {
        String name = newEvent.get("name").toString();
        String value = newEvent.get("value").toString();
        String colo = newEvent.get("colo").toString();
        String retention = newEvent.get("retention").toString();
        String app = newEvent.get("app").toString();
        String gs;
        if (retention.isEmpty()) {
          gs = app + "." + name + "." + colo + "." + suffix + " " + value;
        } else {
          gs = retention + "." + app + "." + name + "." + colo + "." + suffix + " " + value;
        }

        if (gs == null) {
          logger.error("Null string detected");
        }

        logger.debug("graphite string:" + gs);

        // Send the data
        graphiteInterface.send(graphiteInterface.graphiteString(gs));

      } catch (Exception e) {
        // logger.error("Problem with sending metric to graphite: " +
        // newEvent.toString());
      }

    }
  }
```