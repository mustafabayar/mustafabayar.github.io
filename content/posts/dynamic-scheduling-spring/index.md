---
title: "Dynamic Task Scheduling with Spring"
date: 2018-09-08T08:06:25+06:00
description: How to create custom scheduler in Spring
menu:
  sidebar:
    name: Dynamic Task Scheduling
    identifier: dynamic-task-scheduling
    weight: 50
tags: ["Java", "Spring"]
categories: ["Basic"]
---

### Summary

Spring makes it very easy to schedule a job to run periodically. All we need to do is to put **[@Scheduled](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/Scheduled.html)** annotation above the method and provide the necessary parameters such as fixedRate or cron expression. But when it comes to change this fixedRate on the fly, `@Scheduled` annotation is not enough. Ultimately, what I wanted to do was periodically load my configuration table from a database. But the `fixedRate` which indicates how frequently I will load this table is also stored in the very same database. So what I wanted to do was reading this value from a database and schedule the task according to it. Whenever the value changes, the next execution time should change with it too.
Before going into next step, I also created a repository for all the code in this tutorial to show how this scheduling works. You can find the example code in [my Github page](https://github.com/mustafabayar/java-dynamic-scheduling-tutorial)
Also at the end I will add an alternative way for scheduling with exact date and a way to start the scheduler from external service (like controller).
Please check the above repository for various scheduling examples.

### Loading the value from properties file

First of all, in `@Scheduled` annotation you can only use constant values. To use Spring’s Scheduler, you need to put `@EnableScheduling` annotation above any of your class. I prefer my Main Application class for that purpose, but any of the classes should work. You can retrieve the value for this scheduler from your properties file. Such as;

```
@Scheduled(fixedRateString = "${scheduler.configurationLoadRate}")
public void loadConfigurations() {
...
}
```
``scheduler.configurationLoadRate`` is the property I have defined in my property file.

```
scheduler.configurationLoadRate=3600000
```
But this brings another problem. Whenever I want to change this value, I have to restart the application, which is not very convenient. I wanted to inject a value from a database but we can’t store a value in a variable and use in the annotation because it only accepts constant values. Later I have discovered a way to use values from a database;

### Loading the value from a database

First I am creating a bean which retrieves data from the database whenever it is called. And then giving this bean to my ``@Scheduled`` annotation using **[SpEL](https://docs.spring.io/spring-framework/reference/core/expressions.html)**, so-called **Spring Expression Language**.

```
@Bean
public String getConfigRefreshValue() {
   return configRepository.findOne(Constants.CONFIG_KEY_REFRESH_RATE).getConfigValue();
}
.
.
.
@Scheduled(fixedRateString = "#{@getConfigRefreshValue}")
public void loadConfigurations() {
...
}
```
This works like a charm but Houston, we have a problem. This ``@Scheduled`` annotation only looks at the fixedRate once and never looks at it again. So even if the value changes, it doesn’t care. I mean if all you want is to retrieve this data from a database, you can go with this solution. But I realised that dynamic task scheduling with Spring can not be done by ``@Scheduled`` annotation. So after some search I decided to create my own Scheduler Service that implements **SchedulerConfigurer** which successfully changed the rate whenever the data changes. You can find the solution below.
```
import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.Trigger;
import org.springframework.scheduling.TriggerContext;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.stereotype.Service;

@Service
public class SchedulerService implements SchedulingConfigurer {

    @Autowired
    ConfigurationService    configurationService;
   
    @Bean
    public TaskScheduler poolScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setThreadNamePrefix("ThreadPoolTaskScheduler");
        scheduler.setPoolSize(1);
        scheduler.initialize();
        return scheduler;
    }

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(poolScheduler());
        taskRegistrar.addTriggerTask(new Runnable() {
            @Override
            public void run() {
                // Do not put @Scheduled annotation above this method, we don't need it anymore.
                configurationService.loadConfigurations();
            }
        }, new Trigger() {
            @Override
            public Date nextExecutionTime(TriggerContext triggerContext) {
                Calendar nextExecutionTime = new GregorianCalendar();
                Date lastActualExecutionTime = triggerContext.lastActualExecutionTime();
                nextExecutionTime.setTime(lastActualExecutionTime != null ? lastActualExecutionTime : new Date());
                nextExecutionTime.add(Calendar.MILLISECOND, Integer.parseInt(configurationService.getConfiguration(Constants.CONFIG_KEY_REFRESH_RATE_CONFIG).getConfigValue()));
                return nextExecutionTime.getTime();
            }
        });
    }

}
```
We can also write the same function with lambda expressions which will be more compact;
```
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(poolScheduler());
        taskRegistrar.addTriggerTask(() -> configurationService.loadConfigurations(), t -> {
            Calendar nextExecutionTime = new GregorianCalendar();
            Date lastActualExecutionTime = t.lastActualExecutionTime();
            nextExecutionTime.setTime(lastActualExecutionTime != null ? lastActualExecutionTime : new Date());
            nextExecutionTime.add(Calendar.MILLISECOND,
                    Integer.parseInt(configurationService.getConfiguration(Constants.CONFIG_KEY_REFRESH_RATE_CONFIG).getConfigValue()));
            return nextExecutionTime.getTime();
        });
    }
```
I tried to add cancelling and re-activating feature to the Scheduler. With little tweak to above code we can achieve it, but I am not sure if it is the optimal solution or not, so use it at your own risk:
```
package com.mbcoder.scheduler.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.stereotype.Service;

import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;
import java.util.concurrent.ScheduledFuture;

/**
 * Alternative version for DynamicScheduler
 * This one should support everything the basic dynamic scheduler does,
 * and on top of it, you can cancel and re-activate the scheduler.
 */
@Service
public class CancellableScheduler implements SchedulingConfigurer {

    private static Logger LOGGER = LoggerFactory.getLogger(DynamicScheduler.class);

    ScheduledTaskRegistrar scheduledTaskRegistrar;

    ScheduledFuture future;

    @Bean
    public TaskScheduler poolScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setThreadNamePrefix("ThreadPoolTaskScheduler");
        scheduler.setPoolSize(1);
        scheduler.initialize();
        return scheduler;
    }

    // We can have multiple tasks inside the same registrar as we can see below.
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        if (scheduledTaskRegistrar == null) {
            scheduledTaskRegistrar = taskRegistrar;
        }
        if (taskRegistrar.getScheduler() == null) {
            taskRegistrar.setScheduler(poolScheduler());
        }

        future = taskRegistrar.getScheduler().schedule(() -> scheduleFixed(), t -> {
            Calendar nextExecutionTime = new GregorianCalendar();
            Date lastActualExecutionTime = t.lastActualExecutionTime();
            nextExecutionTime.setTime(lastActualExecutionTime != null ? lastActualExecutionTime : new Date());
            nextExecutionTime.add(Calendar.SECOND, 7);
            return nextExecutionTime.getTime();
        });

        // or cron way
        taskRegistrar.addTriggerTask(() -> scheduleCron(repo.findById("next_exec_time").get().getConfigValue()), t -> {
            CronTrigger crontrigger = new CronTrigger(repo.findById("next_exec_time").get().getConfigValue());
            return crontrigger.nextExecutionTime(t);
        });
    }

    public void scheduleFixed() {
        LOGGER.info("scheduleFixed: Next execution time of this will always be 5 seconds");
    }

    public void scheduleCron(String cron) {
        LOGGER.info("scheduleCron: Next execution time of this taken from cron expression -> {}", cron);
    }

    /**
     * @param mayInterruptIfRunning {@code true} if the thread executing this task
     * should be interrupted; otherwise, in-progress tasks are allowed to complete
     */
    public void cancelTasks(boolean mayInterruptIfRunning) {
        LOGGER.info("Cancelling all tasks");
        future.cancel(mayInterruptIfRunning); // set to false if you want the running task to be completed first.
    }

    public void activateScheduler() {
        LOGGER.info("Re-Activating Scheduler");
        configureTasks(scheduledTaskRegistrar);
    }

}
```
We don’t have to keep the reference to future, we can add and remove jobs from external service like;
```
package com.mbcoder.scheduler.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.concurrent.ScheduledFuture;

@Service
public class ExternalScheduler implements SchedulingConfigurer {

    private static Logger LOGGER = LoggerFactory.getLogger(ExternalScheduler.class);

    ScheduledTaskRegistrar scheduledTaskRegistrar;

    Map<String, ScheduledFuture> futureMap = new HashMap<>();

    @Bean
    public TaskScheduler poolScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setThreadNamePrefix("ThreadPoolTaskScheduler");
        scheduler.setPoolSize(1);
        scheduler.initialize();
        return scheduler;
    }

    // Initially scheduler has no job
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        if (scheduledTaskRegistrar == null) {
            scheduledTaskRegistrar = taskRegistrar;
        }
        if (taskRegistrar.getScheduler() == null) {
            taskRegistrar.setScheduler(poolScheduler());
        }
    }

    public boolean addJob(String jobName) {
        if (futureMap.containsKey(jobName)) {
            return false;
        }

        ScheduledFuture future = scheduledTaskRegistrar.getScheduler().schedule(() -> methodToBeExecuted(), t -> {
            Calendar nextExecutionTime = new GregorianCalendar();
            Date lastActualExecutionTime = t.lastActualExecutionTime();
            nextExecutionTime.setTime(lastActualExecutionTime != null ? lastActualExecutionTime : new Date());
            nextExecutionTime.add(Calendar.SECOND, 5);
            return nextExecutionTime.getTime();
        });

        configureTasks(scheduledTaskRegistrar);
        futureMap.put(jobName, future);
        return true;
    }

    public boolean removeJob(String name) {
        if (!futureMap.containsKey(name)) {
            return false;
        }
        ScheduledFuture future = futureMap.get(name);
        future.cancel(true);
        futureMap.remove(name);
        return true;
    }

    public void methodToBeExecuted() {
        LOGGER.info("methodToBeExecuted: Next execution time of this will always be 5 seconds");
    }

}
```
Since some of you asked for the code of my ConfigurationService, I decided to post the code here. Below you can find the implementation of Configuration model, ConfigRepository, ConfigurationService and Constants:
```
@Entity
public class Configuration {

    @Id
    @Size(max = 128)
    String  configKey;

    @Size(max = 512)
    @NotNull
    String  configValue;

    public Configuration() {
    }

    public Configuration(String configKey, String configValue) {
        this.configKey = configKey;
        this.configValue = configValue;
    }

    public String getConfigKey() {
        return configKey;
    }

    public void setConfigKey(String configKey) {
        this.configKey = configKey;
    }

    public String getConfigValue() {
        return configValue;
    }

    public void setConfigValue(String configValue) {
        this.configValue = configValue;
    }

}
```
```
public interface ConfigRepository extends JpaRepository<Configuration, String> {

}
```
```
/**
 * ConfigurationService is responsible for loading and checking configuration parameters.
 *
 * @author mbcoder
 *
 */
@Service
public class ConfigurationService {

    private static final Logger         LOGGER  = LoggerFactory.getLogger(ConfigurationService.class);

    ConfigRepository                    configRepository;

    private Map<String, Configuration>  configurationList;

    private List<String>                mandatoryConfigs;

    @Autowired
    public ConfigurationService(ConfigRepository configRepository) {
        this.configRepository = configRepository;
        this.configurationList = new ConcurrentHashMap<>();
        this.mandatoryConfigs = new ArrayList<>();
        this.mandatoryConfigs.add(Constants.CONFIG_KEY_REFRESH_RATE_CONFIG);
        this.mandatoryConfigs.add(Constants.CONFIG_KEY_REFRESH_RATE_METRIC);
        this.mandatoryConfigs.add(Constants.CONFIG_KEY_REFRESH_RATE_TOKEN);
        this.mandatoryConfigs.add(Constants.CONFIG_KEY_REFRESH_RATE_USER);
    }

    /**
     * Loads configuration parameters from Database
     */
    @PostConstruct
    public void loadConfigurations() {
        LOGGER.debug("Scheduled Event: Configuration table loaded/updated from database");
        StringBuilder sb = new StringBuilder();
        sb.append("Configuration Parameters:");
        List<Configuration> configs = configRepository.findAll();
        for (Configuration configuration : configs) {
            sb.append("\n" + configuration.getConfigKey() + ":" + configuration.getConfigValue());
            this.configurationList.put(configuration.getConfigKey(), configuration);
        }
        LOGGER.debug(sb.toString());

        checkMandatoryConfigurations();
    }

    public Configuration getConfiguration(String key) {
        return configurationList.get(key);
    }

    /**
     * Checks if the mandatory parameters are exists in Database
     */
    public void checkMandatoryConfigurations() {
        for (String mandatoryConfig : mandatoryConfigs) {
            boolean exists = false;
            for (Map.Entry<String, Configuration> pair : configurationList.entrySet()) {
                if (pair.getKey().equalsIgnoreCase(mandatoryConfig) && !pair.getValue().getConfigValue().isEmpty()) {
                    exists = true;
                }
            }
            if (!exists) {
                String errorLog = String.format("A mandatory Configuration parameter is not found in DB: %s", mandatoryConfig);
                LOGGER.error(errorLog);
            }
        }

    }
}
```
Alternatively we can also do the scheduling by giving the exact date, in that case we don’t need to know previous execution time. For example:
```
// startDate and endDate are only calendar date and time indicates at which hour/minute of the day.
public void scheduleAt(LocalDate startDate, LocalDate endDate, LocalTime time) {
    LocalDate now = LocalDate.now();
    if (now.isBefore(endDate)) {
        if (now.isBefore(startDate)) {
            now = startDate;
        }
        LocalDateTime current = now.atTime(time);
        ZoneId zone = ZoneId.of("Europe/Berlin");
        ZoneOffset zoneOffSet = zone.getRules().getOffset(current);
        Instant nextRunTime = current.toInstant(zoneOffSet);
        poolScheduler().schedule(() -> realMethod(), nextRunTime);
    }
}

public void realMethod() {
    // This is your real code to be scheduled
}
```
### Final Words
You can expand this solution to run this every day for example, and you can load start/end dates from database.

Don’t forget to check my [Github repository](https://github.com/mustafabayar/java-dynamic-scheduling-tutorial) to have a better understanding of how this code looks like, and also if this helped you, feel free to give the repository a star 🙂

All this solutions are my own interpretation. For production level usage, you may want to use a scheduling library such as Jesque.