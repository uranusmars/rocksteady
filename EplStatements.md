## Introduction ##

Here we are going to cover adding your own statement to Rocksteady.  EPL(Esper Processing Language) is a language used in [Esper](http://esper.codehaus.org/), Rocksteady's choice of complex event processing engine.  EPL looks similar to SQL with "select", "from", or "where" in its syntax, the biggest difference between EPL and SQL is the data source.  In SQL, query is ran against data already stored in a database.  In EPL, query is ran against data as they arrive.

Case in point:

**SELECT `*` FROM metric WHERE value > 80**

|SQL|Return all rows from a table named "metric" where a field named "value" has a value larger than 80.  Data is in database.|
|:--|:------------------------------------------------------------------------------------------------------------------------|
|EPL|Whenever new "metric" POJO is fed into Esper, query is executed and it will return all POJO(plain old java object) that has attribute named "value" larger than 80|

This, is how CEP process data in real time.  Because EPL processes  data as they come in, there are a few additional language feature that's applicable to EPL such as setting time window of the data.

BTW, a POJO is just a simple java class with properties and getter/setter for those properties.

Here's a simple POJO called metric.
```
public class Metric {

	private String name;
	private Double value;

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Double getValue() {
		return value;
	}

	public void setValue(Double value) {
		this.value = value;
	}
}
```

## Details ##

When metrics come from application into MQ, Rocksteady consumes and parses the metrics into POJO(plain old java object) which is then feed into Esper CEP engine.  Esper will then run all the statements against this new pojo and what comes out depend on the statement.  It could be an aggregation statement which just add the value to existing value, or it could be a threshold statement which might trigger an email or Nagios alert or it could be a correlation statement which needs further processing by other statement.

Assuming we are using [default metric convention](MetricFormat.md),

`retention.app_name.component+.colo.hostname value unix_timestamp`

these are the field name that's available to use in query.:

```
  private String original;
  private String retention = "";
  private String app = "";
  private String name;
  private String colo = "";
  private String hostname = "";
  private Double value;
  private Double timestamp;
```

So for metric string

`1min.juicer.system.cpu.prct_idle.dc1.pi101 82 1283192317`

|original|`1min.juicer.system.cpu.prct_idle.dc1.pi101 82 1283192317`|
|:-------|:---------------------------------------------------------|
|retention|1min                                                      |
|app     |juicer                                                    |
|name    |system.cpu.prct\_idle                                     |
|colo    |dc1                                                       |
|hostname|pi101                                                     |
|value   |82                                                        |
|timestamp|1283192317                                                |

A statement could look something like this: (From [eplStatement.xml](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/spring/eplStatements.xml))

```
  <bean class="com.admob.rocksteady.router.cep.StatementBean" id="metricAveraged">
  <constructor-arg value="SELECT name, colo, avg(value)  as value, retention, app
                          FROM Metric(app='admixer' or app='ustore').std:groupby(colo,name,app).win:time_batch(${cepTimeBatchEverest} sec)
                          GROUP BY name, colo, retention, app                                                                                                  
                          "/>
    <property name="listeners">
     <list>
       <bean class="com.admob.rocksteady.reactor.Graphite">
           <property name="suffix" value="averaged"/>
       </bean>    
     </list>
    </property>
  </bean>
```

For none Java folks, bean is like little piece of code that can be used later in other places.  Instead of coding actual Java, you can see from the statement above, we initiate a new com.admob.rocksteady.router.cep.StatementBean class with value in constructor-arg, gave it an 'ID', then set one of its property "listeners" with another bean.

The id "metricAveraged" is then used in [epl.xml](http://rocksteady.googlecode.com/svn/trunk/src/main/resources/META-INF/spring/epl.xml) to tell Esper what statements to load.

For detail documentation on EPL, please refer to [Esper documentation](http://esper.codehaus.org/esper/documentation/documentation.html).

Let's look at what this statement does:
```
SELECT name, colo, avg(value)  as value, retention, app
FROM Metric(app='admixer' or app='ustore').std:groupby(colo,name,app).win:time_batch(${cepTimeBatchEverest} sec)
GROUP BY name, colo, retention, app  
```

This EPL look for metrics with app name equal to 'admixer' or 'ustore', then group them by colo, name, and app and set a time window of ${cepTimeBatchEverest} seconds for each unique group by.  Then at output time, we have another GROUP BY so rows in time batch window is output once.  An aggregate function "avg()" is used to calculate average value of each metric per colo and app.  Confused?

Okey, I think anyone who's familiar with SQL was fine with the statement until they see

`Metric(app='admixer' or app='ustore').std:groupby(colo,name,app).win:time_batch(${cepTimeBatchEverest} sec)`

and went WTF!

Those are the subtle differences between SQL and EPL.  EPL was meant to process data as they arrive, so there are a few technique to limit number of POJO that query needs to run against.  First thing is this syntax call 'filter':

`Metric(app='admixer' or app='ustore')`

Filter is ran before POJO enters the statement and it's cheap, think of it as limiting input.  The same statement could have been rewritten using WHERE clause like "WHERE app = 'admixer' or app = 'ustore'", but WHERE clause is used to limit output rows which is more intensive.

Then statement is followed by

`std:groupby(colo,name,app)`

You can read the difference between this group by and sql look-alike group by, from [Eper doc's groupby section](http://esper.codehaus.org/esper-3.5.0/doc/reference/en/html_single/index.html#epl-group-by-versus-view).

Then follow by:

`win:time_batch(${cepTimeBatchEverest} sec)`

${cepTimeBatchEverest} is "60", you can find it in rocksteady.properties file.  This filter instruct Esper to gather minutely data set.  Every 60 seconds, current POJO in dataset is outputted and discarded, and newly arrived data is stored until the next batch interval.  There are many different data window scheme, it could be time based, it could be size based, read about it on [this section of Esper doc](http://esper.codehaus.org/esper-3.5.0/doc/reference/en/html_single/index.html#processingmodel_time_window).

Last piece on that statement is

```
    <property name="listeners">
     <list>
       <bean class="com.admob.rocksteady.reactor.Graphite">
           <property name="suffix" value="averaged"/>
       </bean>    
     </list>
    </property>
```

If statement needs to output anything, we use a "listener" to listen for data that statement captures.  In this statement, the minutely average of all "admixer" and "ustore" metrics per colo, per application is sent out using Graphite and a suffix "averaged" is added to the original metric name.  Note: you can have multiple listeners for one statement.

For more information on available listeners or write your own, refer to [Epl Listener Wiki](EplListener.md).