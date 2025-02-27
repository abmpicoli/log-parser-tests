# Objectives:
This project is a proof-of-concept if 
logs from a docker container output can be captured in a reliable way using 
adobe log parser ('com.adobe.campaign.tests:log-parser:1.11.2')


The actors:

* An operator watches a running solution, checking inbound and outbound data, and make quick inspections
  about the application health.
* A solution developer uses a third party framework to develop custom tailored solutions for specific
  customer use cases. Examples, collect data from service now and send to kafka. Operators may invoke solution developers
  when they find issues with the solution.
* A framework developer maintains the third party framework the solution developer uses.
* The data owner owns 

The use cases:

* UC0001 As an operator I want to filter out any message that is not relevant to the general application health:
  I don't want to see initialization options, etc.: I want to see which records were processed with success,
  which failed, and any big ugly errors I see there.
* UC0002 As a solution developer, when the operator say they have an issue with record X, 
  I want to be able to track the detailed history around the time of that record, to understand why there was an issue, 
  and at which process step.
* UC0003 As a solution developer, when I'm testing a solution, I want to see if there are any issues with the 
  configuration during startup.
* UC0004 As the framework developer, I need to see all logs in detail, to help diagnose issues.
* UC0005 As the framework developer, I must be able to collect logs specific to a third party solution my framework 
  depends on.
* UC0006 As any role, I may collect huge amounts of data in heterogeneous formats, and I want to be able to filter 
  that data by date and further data qualifiers. 
  * For example, in a log4j2 log we would have log level, thread and category.
  * For a typical http access log we would have a method (GET/POST/PUT, etc) a status code remote ip and local url.
* UC0007 As the data owner I demand that sensitive data in general is not generally disclosed, except on the specific 
  cases I authorize it, and for as short a time as possible.


The challenge in a typical java tomcat container log output is that, if a mix of
shellscript, accessory apps and a web container is used,
the log output will be widely inconsistent: some echo commands, mixed with
raw utilities output, mixed with tomcat output,
mixed with web-application outputs, mixed with dependency log outputs and
random syserr / sysout streams.  

The resource at [/src/test/sample_log.txt](/src/test/sample_log.txt) shows an example 
of what I'm talking about. In there you will find:

Scripts writing log in a specific format. Example:

```
>>> 2025-02-13 18:44:11.000 ou=the_ou_keyDEV_oauth_basic_DETAILED Adjusting permissions
changed ownership of '/var/is/deployment/tomcat/conf/context.xml' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/conf/server.xml' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/conf' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/lib/db2jcc4.jar' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/lib' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/bin/setenv.sh' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/bin' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/log4j2/log4j2.xml' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat/log4j2' from root:root to tomcat:tomcat
changed ownership of '/var/is/deployment/tomcat' from root:root to tomcat:tomcat
```

Accessory applications, like open telemetry, writing logs in another format:

```
2025-02-13 18:44:19,909 ou=the_ou_keyDEV_oauth_basic_DETAILED INFO [ open-telemetry | open-telemetry ] 2025-02-13T18:44:19.908Z	info	service@v0.119.0/service.go:186	Setting up own telemetry...
2025-02-13 18:44:19,909 ou=the_ou_keyDEV_oauth_basic_DETAILED INFO [ open-telemetry | open-telemetry ] 2025-02-13T18:44:19.909Z	info	service@v0.119.0/service.go:252	Starting otelcol...	{"Version": "0.119.0", "NumCPU": 8}
2025-02-13 18:44:19,909 ou=the_ou_keyDEV_oauth_basic_DETAILED INFO [ open-telemetry | open-telemetry ] 2025-02-13T18:44:19.909Z	info	extensions/extensions.go:39	Starting extensions...
2025-02-13 18:44:19,910 ou=the_ou_keyDEV_oauth_basic_DETAILED INFO [ open-telemetry | open-telemetry ] 2025-02-13T18:44:19.910Z	info	otlpreceiver@v0.119.0/otlp.go:112	Starting GRPC server	{"kind": "receiver", "name": "otlp", "data_type": "metrics", "endpoint": "127.0.0.1:4317"}
2025-02-13 18:44:19,910 ou=the_ou_keyDEV_oauth_basic_DETAILED INFO [ open-telemetry | open-telemetry ] 2025-02-13T18:44:19.910Z	info	service@v0.119.0/service.go:275	Everything is ready. Begin running and processing data.
```

Tomcat itself, writing logs in yet another format.

```
2025-02-13 18:44:20,617 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] Server version name:   Apache Tomcat/9.0.99
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] Server built:          Feb 4 2025 20:08:08 UTC
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] Server version number: 9.0.99.0
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] OS Name:               Linux
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] OS Version:            5.15.167.4-microsoft-standard-WSL2
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] Architecture:          amd64
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] Java Home:             /opt/java/openjdk
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] JVM Version:           21.0.6+7-LTS
2025-02-13 18:44:20,620 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] JVM Vendor:            Eclipse Adoptium
2025-02-13 18:44:20,621 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] CATALINA_BASE:         /usr/local/tomcat
```

And finally, the two web-applications running inside tomcat:

```
2025-02-13 18:44:21,536 ou=the_ou_keyDEV_oauth_basic_DETAILED APP INFO  [ main | n.k.i.b.Bridge ] Initializing Bridge Vendor:Kyndryl Integration Services Title:bridge Version:11.1.1-java21
2025-02-13 18:44:21,541 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081A evaluate invoked
2025-02-13 18:44:21,545 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081B Checking trace verbosity settings for net.kyndryl.is.bridge.verbose.config : Show verbose configuration: raw configuration output, from where a configuration is coming from
2025-02-13 18:44:21,546 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081C_B Extra verbose logging for net.kyndryl.is.bridge.verbose.config is being deactivated
2025-02-13 18:44:21,546 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081D appender console added to net.kyndryl.is.bridge.verbose.config
2025-02-13 18:44:21,547 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081A evaluate invoked
2025-02-13 18:44:21,547 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081B Checking trace verbosity settings for net.kyndryl.is.bridge.verbose.xml : Show verbose transforms evaluations
2025-02-13 18:44:21,547 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081C_B Extra verbose logging for net.kyndryl.is.bridge.verbose.xml is being deactivated
2025-02-13 18:44:21,548 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081D appender console added to net.kyndryl.is.bridge.verbose.xml
2025-02-13 18:44:21,548 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081A evaluate invoked
2025-02-13 18:44:21,548 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081B Checking trace verbosity settings for net.kyndryl.is.bridge.verbose.xpath : Verbose information about xpath evaluation
2025-02-13 18:44:21,548 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081C_B Extra verbose logging for net.kyndryl.is.bridge.verbose.xpath is being deactivated
2025-02-13 18:44:21,549 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081D appender console added to net.kyndryl.is.bridge.verbose.xpath
2025-02-13 18:44:21,549 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081A evaluate invoked
2025-02-13 18:44:21,549 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.Verbosity ] VBS0081B Checking trace verbosity settings for net.kyndryl.is.bridge.verbose.sse : Verbose information about SSE event handling
2025-02-13 18:44:22,212 ou=the_ou_keyDEV_oauth_basic_DETAILED APP TRACE [ main | n.k.i.c.m.ClasspathURLProperty ] CUP0043E using the ClasspathURLProperty classloader ParallelWebappClassLoader
  context: sn-pr-src-in
  delegate: false
----------> Parent Classloader:
java.net.URLClassLoader@2b40ff9c
 for fetching resources, and ClasspathConfigurationOverride ParallelWebappClassLoader
  context: sn-pr-src-in
  delegate: false
----------> Parent Classloader:
java.net.URLClassLoader@2b40ff9c

```

The challenges: 

* There are single message logs that spans multiple lines.
* The separation between log entries can be heterogeneous: a log from open telemetry intermixed
  with a multiline log from the main application, for example.
* Applications running in a multithreaded environment may have logs belonging to different 
  records intertwined. Due to UC0002, I should be able to collect only logs belonging to that thread.

UPDATE:
=======

After careful considerations, I've considered the adobe log parser solution unfit for our purposes:

1) It assumes homogeneous data in single line entities.
2) It keeps all findings in memory.
3) The parsed relies on the presence of specific separators: with heterogeneous formats, the separators may change.

These two reasons makes it unfit for UC0006: huge heterogeneous data content.

Thinking my own architecture:
=============================

A log line is composed of a header + fields + content.

Once a header is found, a tokenizer should find syntax aspects in an abstract syntax tree, until parsing is no longer found,
to find fields. When the syntax no longer applies, the content is actual message contents, until another header is found.

Using a minor variant of w3c notation https://www.w3.org/TR/REC-xml/#sec-notation

Extra notation:

? <<comment>> ?  : the notation for a human-readable parse specification

`>>capture=fieldname <<` specifies a capture field 

Example:

`2025-02-13 18:44:20,617 ou=the_ou_keyDEV_oauth_basic_DETAILED TOMCAT INFO  [ main | o.a.c.s.VersionLoggerListener ] Server version name:   Apache Tomcat/9.0.99`

The header would be 

`newline ::= (? start of a file ?) | (\n|\r|\n\r)`

`digit ::= [0-9]`
`wordchar ::= (\w|.|-|_|$)+`
`date ::= digit{4}-digit{2}-digit{2}`
`time ::= ' ' digit{2} ':' digit{2} ':' digit{2}`
`millis ::= [.,]digit+`
`timestamp ::= >>capture=timestamp date time millis? <<`
`header ::= newline timestamp`

The fields would be

`oudef ::= 'ou=' wordchar`
`spc ::= ' '+`
`field ::= spc oudef spc appname spc loglevel spc threadcategory spc`

`appname ::= >>capture=app wordchar<<`
`loglevel ::= >>capture=level wordchar<<`
`threadcategory ::= '[' spc >>capture=thread wordchar << spc '|' spc >>capture=category<< wordchar spc ']'`



`>>> 2025-02-13 18:44:11.000 ou=the_ou_keyDEV_oauth_basic_DETAILED Adjusting permissions`

the header would be

header ::= newline '>>>' spc timestamp  

TODO


