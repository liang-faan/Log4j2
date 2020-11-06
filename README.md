# Log4j2 package as logging masker in Spring boot Jar and share to other projects/modules 
## Step#1 - Create Logger Masking Converter
```
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.logging.log4j.core.LogEvent;
import org.apache.logging.log4j.core.config.plugins.Plugin;
import org.apache.logging.log4j.core.pattern.ConverterKeys;
import org.apache.logging.log4j.core.pattern.LogEventPatternConverter;

@Plugin(name = "LogMaskingConverter", category = "Converter")
@ConverterKeys({"spi", "trscId"}) //Here the converter keys map to below Log4j2.xml %spi 
public class LogMaskingConverter extends LogEventPatternConverter {
    private static final String CREDIT_CARD_REGEX = "([0-9]{16})";
    private static final Pattern CREDIT_CARD_PATTERN = Pattern.compile(CREDIT_CARD_REGEX);
    private static final String CAREDIT_CARD_REPLACEMENT_REGEX = "XXXXXXXXXXXXXXXX";

    private static final String CVV_REGEX = "([0-9]{3})";
    private static final Pattern CVV_PATTERN = Pattern.compile(CVV_REGEX);
    private static final String CVV_REPLACEMENT_REGEX = "+++";

    private static final String SSN_REGEX = "([0-9]{9})";
    private static final Pattern SSN_PATTERN = Pattern.compile(SSN_REGEX);
    private static final String SSN_REPLACEMENT_REGEX = "*********";

    private static final String SVC_NO_REGEX = "([89]\\d{7})\\b";
    private static final Pattern SVC_NO_PATTERN = Pattern.compile(SVC_NO_REGEX);
    private static final String SVC_NO_REPLACEMENT_REGEX = "^\\d{4}";

    private static final String JWT_REGEX = "[Bb]earer[A-Za-z0-9-_=]+\\.[A-Za-z0-9-_=]+\\.?[A-Za-z0-9-_.+/=]*";
    private static final Pattern JWT_PATTERN = Pattern.compile(JWT_REGEX);
    private static final String JWT_REPLACEMENT_REGEX = "****";

    protected LogMaskingConverter(String name, String style) {
        super(name, style);
    }

    public static LogMaskingConverter newInstance(String[] options) {
        return new LogMaskingConverter("spi", Thread.currentThread().getName());
    }

    @Override
    public void format(LogEvent event, StringBuilder outputMessage) {
        String message = event.getMessage().getFormattedMessage();
        String maskedMessage = message;
        try {
            maskedMessage = mask(message);
        } catch (Exception e) {
            System.out.println("Failed While Masking");
            maskedMessage = message;
        }
        outputMessage.append(maskedMessage);

    }

    private String mask(String message) {
        Matcher matcher = null;
        StringBuffer buffer = new StringBuffer();
        
        matcher = SVC_NO_PATTERN.matcher(message);
        maskMatcherRegrex(matcher, buffer, SVC_NO_REPLACEMENT_REGEX);
        message = buffer.toString();
        buffer.setLength(0);

        matcher = JWT_PATTERN.matcher(message);
        maskMatcher(matcher, buffer, JWT_REPLACEMENT_REGEX);

        return buffer.toString();
    }

    private StringBuffer maskMatcherRegrex(Matcher matcher, StringBuffer buffer, String maskStr) {
        while (matcher.find()) {
            matcher.appendReplacement(buffer, matcher.group().replaceAll(maskStr, "***"));
        }
        matcher.appendTail(buffer);
        return buffer;
    }

    private StringBuffer maskMatcher(Matcher matcher, StringBuffer buffer, String maskStr) {
        while (matcher.find()) {
            matcher.appendReplacement(buffer, maskStr);
        }
        matcher.appendTail(buffer);
        return buffer;
    }

}

```

## Step#2 Create Log4j2.xml and put under \src\main\resources
```
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" monitorInterval="30" packages="com.my.package.logger">
    <Properties>
        <Property name="LOG_PATTERN">
            [%d] [%p] [%t] [$${bundle:application:spring.application.name}] [%c{1.}] %spi%n
        </Property>
    </Properties>

    <Appenders>
        <Console name="ConsoleAppender" target="SYSTEM_OUT" follow="true">
            <PatternLayout pattern="${LOG_PATTERN}"/>
        </Console>

        <!-- Rolling File Appender -->
        <RollingFile name="FileAppender" fileName="logs/log4j2-demo.log" filePattern="logs/log4j2-demo-%d{yyyy-MM-dd}-%i.log">
            <PatternLayout>
                <Pattern>${LOG_PATTERN}</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="10MB"/>
            </Policies>
            <DefaultRolloverStrategy max="10"/>
        </RollingFile>

        <!-- SMTP Appender
        <SMTP name="MailAppender"
              subject="Log4j2 Demo [PROD]"
              to="youremail@example.com"
              from="log4j2-demo-alerts@example.com"
              smtpHost="yourSMTPHost"
              smtpPort="587"
              smtpUsername="yourSMTPUsername"
              smtpPassword="yourSMTPPassword"
              bufferSize="1">
            <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout>
                <Pattern>${LOG_PATTERN}</Pattern>
            </PatternLayout>
        </SMTP>
         -->
    </Appenders>

    <Loggers>
        <AsyncLogger name="com.example.log4j2demo" level="${bundle:application:logging.level.root}"
                     additivity="false">
            <AppenderRef ref="ConsoleAppender"/>
<!--            <AppenderRef ref="FileAppender"/>-->
        </AsyncLogger>

        <Root level="${bundle:application:logging.level.root}">
            <AppenderRef ref="ConsoleAppender"/>
<!--            <AppenderRef ref="FileAppender"/>-->
        </Root>
    </Loggers>
</Configuration>
```

## Step#3 Add dependencies in gradle
```
dependencies {
    compile group: 'com.lmax', name: 'disruptor', version: '3.4.2'
    compile 'org.springframework.boot:spring-boot-starter-log4j2'
    annotationProcessor 'org.apache.logging.log4j:log4j-core:2.11.1' # This is to re-generate Log4j2Plugins.dat under META-INF. Without this, springboot bootJar will enconter "Unrecognized format specifier" error
   
  }
```

## Step#4 in other springboot runner projects add below in build.gradle
```
configurations {
    commonFramwork { transitive = false }
    all*.exclude module: 'spring-boot-starter-logging'
}

dependencies {
  commonFramwork '***:***:1.0.0-SNAPSHOT' # Refer to above Log4j2 Artifacts
}

task extractLoggerConfiguration(type: Copy) {
    from zipTree(configurations.commonFramwork.singleFile).matching {
        include '**/log4j2.xml'
    }
    into "$buildDir/resources/main"
    includeEmptyDirs = false
}


compileJava {
    options.compilerArgs = ["-Xlint:unchecked"]
    options.fork = true
    options.forkOptions.executable = 'javac'
    dependsOn(extractLoggerConfiguration)
}
```
