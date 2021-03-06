# Prometheus Job Event Listener for Apache Oozie 


[![Maven metadata URI](https://img.shields.io/maven-metadata/v/http/central.maven.org/maven2/de/m3y/oozie/prometheus/oozie-prometheus-job-event-listener/maven-metadata.xml.svg)](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22de.m3y.oozie.prometheus%22%20AND%20a%3A%22oozie-prometheus-job-event-listener%22)

Exposes [Apache Oozie](https://oozie.apache.org) job metrics to [Prometheus](https://prometheus.io/).

The implementation hooks directly into Oozie by implementing the Oozie [JobEventListener](https://oozie.apache.org/docs/4.2.0/core/apidocs/org/apache/oozie/event/listener/JobEventListener.html).
This has the advantage of a direct instrumentation, versus alternative approaches such as polling database or Oozie API.

For Oozie server metrics (database connections etc.) check out the [Apache Oozie Exporter](https://github.com/marcelmay/apache-oozie-exporter),
or if you run [Apache Oozie 4.3.+](http://oozie.apache.org/docs/4.3.0/release-log.txt) which exposes its internal server metrics to JMX ([OOZIE-2507](https://issues.apache.org/jira/browse/OOZIE-2507)) try the Prometheus [jmx_exporter](https://github.com/prometheus/jmx_exporter).

## Metrics exposed

| Name                                  | Labels          | Description |
|---------------------------------------|-----------------|-------------|
| oozie_workflow_job_duration_seconds   | job_type, app_name, status | Duration of completed (Ooze status SUCCEEDED, KILLED, FAILED,...) job in seconds|
| oozie_workflow_job_state_time_seconds | job_type, app_name, status | Timestamp of completed job state change, including job name and changed state. Can be used for monitoring last successful run.
| oozie_workflow_job_total              | job_type, app_name, status | Count of completed jobs.
| oozie_prometheus_job_listener_purge_duration_seconds | - | Summary tracking purge duration.
| oozie_prometheus_job_listener_purged_items_total | - | Counter tracking number of purged metrics.

Example output:
```
# HELP oozie_workflow_job_state_time_seconds Timestamp of completed job state changes
# TYPE oozie_workflow_job_state_time_seconds gauge
oozie_workflow_job_state_time_seconds{job_type="WORKFLOW_JOB",app_name="test-3",status="SUSPENDED",} 1.512419363921E9
oozie_workflow_job_state_time_seconds{job_type="WORKFLOW_JOB",app_name="test-3",status="KILLED",} 1.512419353909E9
oozie_workflow_job_state_time_seconds{job_type="WORKFLOW_JOB",app_name="test-4",status="SUCCEEDED",} 1.51241937396E9
oozie_workflow_job_state_time_seconds{job_type="WORKFLOW_JOB",app_name="test-4",status="RUNNING",} 1.51241937396E9
oozie_workflow_job_state_time_seconds{job_type="WORKFLOW_JOB",app_name="test-3",status="RUNNING",} 1.512419363921E9
# HELP oozie_workflow_job_total Count of completed job state changes
# TYPE oozie_workflow_job_total counter
oozie_workflow_job_total{job_type="WORKFLOW_JOB",app_name="test-3",status="SUSPENDED",} 1.0
oozie_workflow_job_total{job_type="WORKFLOW_JOB",app_name="test-3",status="KILLED",} 2.0
oozie_workflow_job_total{job_type="WORKFLOW_JOB",app_name="test-4",status="SUCCEEDED",} 2.0
oozie_workflow_job_total{job_type="WORKFLOW_JOB",app_name="test-4",status="RUNNING",} 2.0
oozie_workflow_job_total{job_type="WORKFLOW_JOB",app_name="test-3",status="RUNNING",} 3.0
# HELP oozie_workflow_job_duration_seconds Duration of completed jobs
# TYPE oozie_workflow_job_duration_seconds gauge
oozie_workflow_job_duration_seconds{job_type="WORKFLOW_JOB",app_name="test-3",status="KILLED",} 667.0
oozie_workflow_job_duration_seconds{job_type="WORKFLOW_JOB",app_name="test-4",status="SUCCEEDED",} 0.0
```

## Installing

1) Add JAR with Prometheus Job Event Listener to Oozie WAR

   The shaded JAR already contains all required Prometheus Maven dependencies.
   ```
   cp oozie-prometheus-job-event-listener-VERSION-shaded.jar <OOZIE_SERVER_DIR>/oozie-server/webapps/oozie/WEB-INF/lib
   ```
   Alternatively, add the shaded JAR to the oozie.war and redeploy it (after perform step 3 with web.xml update).
   
2) Configure [Oozie job event listener](https://oozie.apache.org/docs/4.2.0/AG_Install.html#Notifications_Configuration)

   Edit oozie-site.xml and enable event handler service:
   ```xml
   <property>
       <name>oozie.services.ext</name>
       <value>
           ...
           org.apache.oozie.service.EventHandlerService,
       </value>
   </property>
   ```
   Add the Prometheus job event listener to the list of listeners:
   ```xml
   <property>
       <name>oozie.service.EventHandlerService.event.listeners</name>
       <value>de.m3y.oozie.prometheus.PrometheusJobEventListener</value>
   </property>
   ```

3) Expose Prometheus metrics via [simpleclient_servlet](https://github.com/prometheus/client_java/tree/master/simpleclient_servlet) from Oozie web application

   Edit <OOZIE_SERVER_DIR>/oozie-server/webapps/oozie/WEB-INF/web.xml and add the Prometheus MetricServlet:
   ```xml
   <servlet>
        <servlet-name>Prometheus Metrics</servlet-name>
        <servlet-class>io.prometheus.client.exporter.MetricsServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>Prometheus Metrics</servlet-name>
        <url-pattern>/metrics</url-pattern>
    </servlet-mapping>
   ```
   The metrics will be available at http://<OOZIE-SERVER>:11000/oozie/metrics .

   **Note**: After step 1...3 you want to restart Oozie for activating the configuration changes.

4) Add Oozie to Prometheus scraping
   Edit your Prometheus config and add the scrape config:
   ```yaml
   - job_name: 'oozie_events'
     metrics_path: 'oozie/metrics'
     static_configs:
       - targets: ['localhost:11000']
   ```
   **Note**: Replace localhost with the name of your Oozie server, and change the Oozie default port if required.
   
5) Optionally change the default purge timeout (three days) and purge interval (one day) for idle event samples
   by setting a system property via JVM options:
   ```
   -DPrometheusJobEventListener.TimeSeries.Idle.TTL.seconds=259200
   -DPrometheusJobEventListener.Purge.Interval.seconds=86400
   ```
   The listener tracks events in metric time series. So after a job terminates,
   all its metrics should be purged after an idle timeout, to avoid clogging up memory.
   Oozie workflow does have an end state which could trigger purges, but its not guaranteed and sometimes fails.

## Building
```
mvn install
```

## Requirements

* Oozie 4.2.x

## License

Licensed under [Apache 2.0 License](LICENSE)
