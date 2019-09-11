# ubuntu-docker-remote-syslog
Ubuntu docker image to receive system logs from remote server

**Syslog**: Sending Java log4j2 to rsyslog on Ubuntu

Enable rsyslog on receiving Ubuntu Server
Modify ‘/etc/rsyslog.conf’ (uncomment the lines that listen on the port 514 UDP port)
Additionally, add a line defining the template  ‘jsonRfc5424Template’ which will allow us to write the log information as json.

----------
- provides UDP syslog reception, uncomment the two lines below
$ModLoad imudp
$UDPServerRun 514
$UDPServerAddress 127.0.0.1

$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
- add the line below which provides json output
$template $template jsonRfc5424Template,"{\"type\":\"syslog\",\"host\":\"%HOSTNAME%\",\"message\":\"<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg:::json%\"}\n"
----------

Restart the rsyslog service and the log output should go to ‘/var/log/syslog’.

Have our application logs written to their own file in a json format:
Create ‘/etc/rsyslog.d/30-testlog4j.conf’ with the content below which tells syslog to write any syslog messages from ‘testlog4j’ to their own file in json format, and then stop any further processing on the message

----------
if $programname == 'testlog4j' or $syslogtag == 'testlog4j' then /var/log/testlog4j/testlog4j.log;jsonRfc5424Template
& stop
----------

Create the log directory and make sure the syslog port 514 is enabled on the local firewall and restart the rsyslog service
- mkdir -p /var/log/testlog4j
- chown syslog:syslog /var/log/testlog4j
- chmod 755 /var/log/testlog4j
- ufw allow 514/udp
- service rsyslog restart

If the rsyslog service is not started (“ps -A | grep rsyslog”), then errors in the rsyslog configuration can be found by:
- rsyslogd -N1

Validating syslog processing
From the host where the Java application server will actually run, use the standard Ubuntu ‘logger’ utility to send syslog messages via UDP
- logger -p local0.warn -d -n myhost "test message to catchall" -u /ignore/socket
Specify a syslog tag of ‘testlog4j’
- logger -t testlog4j -p local0.warn -d -n myhost "to testlog4j" -u /ignore/socket


------------------------------------------------------------------------------------------

Send log4j2 messages to Syslog

----------
import org.apache.logging.log4j.*;

public class TestLog4j {
    
    private static final Logger logger = LogManager.getLogger(TestLog4j.class);
    
    public static void main(String[] args) throws Exception
      {
         logger.debug("debug message");
         logger.info("info message");
         logger.warn("warn message");
         logger.error("error message");
         try {
             int i = 1/0;
         }catch(Exception exc) {
             logger.error("error message with stack trace",
                     new Exception("I forced this exception",exc));
         }
         logger.fatal("fatal message");
      }    

}
----------

Dependencies: log4j-api -<version>.jar and log4j-core-<version>.jar

log4j2.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn">

<Appenders>
    <Console name="console" target="SYSTEM_OUT">
      <PatternLayout pattern="TOCONSOLE %d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
    
    <!--  OPTION#1: Use standard syslog and add fields with LoggerFields -->
    <Syslog name="syslog" format="RFC5424" host="myhost" port="514"
            protocol="UDP" appName="testlog4j" includeMDC="false" mdcId="testlog4j"
            facility="LOCAL0" enterpriseNumber="18060" newLine="false"
            messageId="Audit">
            <LoggerFields>
                  <KeyValuePair key="thread" value="%t"/>
                  <KeyValuePair key="priority" value="%p"/>
                  <KeyValuePair key="category" value="%c"/>
                  <KeyValuePair key="exception" value="%ex"/>
            </LoggerFields>
                        
    </Syslog>
            
    <!--  OPTION#2: Use socket with explicit pattern -->
    <Socket name="syslogsocket" host="myhost" port="514" protocol="UDP">
          <PatternLayout
        pattern="&lt;134&gt;%d{MMM dd HH:mm:ss} ${hostName} testlog4j: {
              &quot;thread&quot;:&quot;%t&quot;,
              &quot;priority&quot;:&quot;%p&quot;,
              &quot;category&quot;:&quot;%c{1}&quot;,
              &quot;exception&quot;:&quot;%exception&quot;
              }%n"
          />
    </Socket>            
            
</Appenders>

<Loggers>
    <Root level="warn">
      <AppenderRef ref="console"/>
      <AppenderRef ref="syslog"/>
    </Root>
</Loggers>    
  
</Configuration>
```


Using the <Syslog> element, you get the following output:
{"type":"syslog","host":"myhost","message":"<132>1 2016-10-16T21:25:20.931-05:00 myhost testc - Audit [testlog4j@18060 category="TestLog4j" exception="" priority="WARN" thread="main"] warn message"}
{"type":"syslog","host":"myhost","message":"<131>1 2016-10-16T21:25:20.933-05:00 myhost testc - Audit [testlog4j@18060 category="TestLog4j" exception="" priority="ERROR" thread="main"] error message"}
{"type":"syslog","host":"myhost","message":"<131>1 2016-10-16T21:25:20.933-05:00 myhost testc - Audit [testlog4j@18060 category="TestLog4j" exception="java.lang.Exception: I forced this exception#012#011at TestLog4j.main(TestLog4j.java:26)#012Caused by: java.lang.ArithmeticException: / by zero#012#011at TestLog4j.main(TestLog4j.java:23)#012" priority="ERROR" thread="main"] error message with stack trace"}
{"type":"syslog","host":"myhost","message":"<129>1 2016-10-16T21:25:20.936-05:00 myhost testc - Audit [testlog4j@18060 category="TestLog4j" exception="" priority="FATAL" thread="main"] fatal message"}

While the <Socket> produces the following:
{"type":"syslog","host":"myhost","message":"<134>1 2016-10-16T21:28:31-05:00 myhost testc - - -  {         \"thread\":\"main\",         \"priority\":\"WARN\",         \"category\":\"TestLog4j\",         \"exception\":\"\"         }"}
{"type":"syslog","host":"myhost","message":"<134>1 2016-10-16T21:28:31-05:00 myhost testc - - -  {         \"thread\":\"main\",         \"priority\":\"ERROR\",         \"category\":\"TestLog4j\",         \"exception\":\"\"         }"}
{"type":"syslog","host":"myhost","message":"<134>1 2016-10-16T21:28:31-05:00 myhost testc - - -  {         \"thread\":\"main\",         \"priority\":\"ERROR\",         \"category\":\"TestLog4j\",         \"exception\":\" java.lang.Exception: I forced this exception#012#011at TestLog4j.main(TestLog4j.java:26)#012Caused by: java.lang.ArithmeticException: / by zero#012#011at TestLog4j.main(TestLog4j.java:23)#012\"         }"}
{"type":"syslog","host":"myhost","message":"<134>1 2016-10-16T21:28:31-05:00 myhost testc - - -  {         \"thread\":\"main\",         \"priority\":\"FATAL\",         \"category\":\"TestLog4j\",         \"exception\":\"\"         }"}

------------------------------------------------------------------------------------------

Syslog: Sending Java SLF4J/Logback to Syslog

Send slf4j2 messages to Syslog

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class TestLogback {
    
    static final Logger logger = LoggerFactory.getLogger(TestLogback.class);
    
    public static void main(String[] args) throws Exception
      {
         logger.debug("debug message slf4j");
         logger.info("info message slf4j");
         logger.warn("warn message slf4j");
         logger.error("error message slf4j");
         try {
             int i = 1/0;
         }catch(Exception exc) {
             logger.error("error message with stack trace slf4j",
                     new Exception("I forced this exception",exc));
         }
      }    

}

Dependencies: slf4j-api-<version>.jar, logback-core-<version>.jar, and logback-classic-<version>.jar

logback.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="true">
 
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <layout class="ch.qos.logback.classic.PatternLayout">
      <Pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</Pattern>
    </layout>
  </appender>
  
    <appender name="SYSLOG" class="ch.qos.logback.classic.net.SyslogAppender">
        <syslogHost>myhost</syslogHost>
        <facility>LOCAL0</facility>
        <port>514</port>
        <!-- include %exception in message instead of allowing default multiline stack trace -->
        <throwableExcluded>true</throwableExcluded>
        <suffixPattern>testlogback %m thread:%t priority:%p category:%c exception:%exception</suffixPattern> 
    </appender>  
  
  <root level="debug">
    <appender-ref ref="STDOUT" />
    <appender-ref ref="SYSLOG" />
  </root>
</configuration>
```

The raw data sent will look something like below:

----------
<135>Oct 17 06:37:55 myhost testlog4j debug message slf4j thread:main priority:DEBUG category:TestLogback exception:
<134>Oct 17 06:37:55 myhost testlog4j info message slf4j thread:main priority:INFO category:TestLogback exception:
<132>Oct 17 06:37:55 myhost testlog4j warn message slf4j thread:main priority:WARN category:TestLogback exception:
<131>Oct 17 06:37:55 myhost testlog4j error message slf4j thread:main priority:ERROR category:TestLogback exception:
<131>Oct 17 06:37:55 myhost testlog4j error message with stack trace slf4j thread:main priority:ERROR category:TestLogback exception:java.lang.Exception: I forced this exception at TestLogback.main(TestLogback.java:20) Caused by: java.lang.ArithmeticException: / by zero at TestLogback.main(TestLogback.java:17)
----------
