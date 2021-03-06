# Querying Application Load Balancer Logs<a name="application-load-balancer-logs"></a>

An Application Load Balancer is a load balancing option for Elastic Load Balancing that enables traffic distribution in a microservices deployment using containers\. Querying Application Load Balancer logs allows you to see the source of traffic, latency, and bytes transferred to and from Elastic Load Balancing instances and backend applications\.

Before you begin, [enable access logging](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html#enable-access-logging) for Application Load Balancer logs to be saved to your Amazon S3 bucket\.
+  [Creating the Table for ALB Logs](#create-alb-table) 
+  [Example Queries for ALB logs](#query-alb-logs-examples) 

## Creating the Table for ALB Logs<a name="create-alb-table"></a>

1. Copy and paste the following DDL statement into the Athena console, and modify values in `LOCATION 's3://your-alb-logs-directory/AWSLogs/<ACCOUNT-ID>/elasticloadbalancing/region'`\. 

   Create the `alb_logs` table as follows\.
**Note**  
This query includes all fields present in the list of current Application Load Balancer [Access Log Entries](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html#access-log-entry-format)\. In addition, it also includes a table column `new_field` at the end, reserved as a placeholder for any future additions to the ALB logs\. Having this field in your query does not break it\. The regular expression in the SerDe properties ignores this field if your logs don't have it\. When new fields are introduced in ALB log entries, they are added to the end of the log entry\. You should ignore any fields at the end of the log entry that you were not expecting\. 

   ```
   CREATE EXTERNAL TABLE IF NOT EXISTS alb_logs (
       type string,
       time string,
       elb string,
       client_ip string,
       client_port int,
       target_ip string,
       target_port int,
       request_processing_time double,
       target_processing_time double,
       response_processing_time double,
       elb_status_code string,
       target_status_code string,
       received_bytes bigint,
       sent_bytes bigint,
       request_verb string,
       request_url string,
       request_proto string,
       user_agent string,
       ssl_cipher string,
       ssl_protocol string,
       target_group_arn string,
       trace_id string,
       domain_name string,
       chosen_cert_arn string,
       matched_rule_priority string,
       request_creation_time string,
       actions_executed string,
       redirect_url string,
       new_field string
       )
       ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
       WITH SERDEPROPERTIES (
       'serialization.format' = '1',
       'input.regex' = 
   '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) ([^ ]*) (- |[^ ]*)\" \"([^\"]*)\" ([A-Z0-9-]+) ([A-Za-z0-9.-]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^\"]*)\" ([-.0-9]*) ([^ ]*) \"([^\"]*)\"($| \"[^ ]*\")(.*)')
       LOCATION 's3://your-alb-logs-directory/AWSLogs/<ACCOUNT-ID>/elasticloadbalancing/region';
   ```

1. Run the query in the Athena console\. After the query completes, Athena registers the `alb_logs` table, making the data in it ready for you to issue queries\.

## Example Queries for ALB logs<a name="query-alb-logs-examples"></a>

The following query counts the number of HTTP GET requests received by the load balancer grouped by the client IP address:

```
SELECT COUNT(request_verb) AS
 count,
 request_verb,
 client_ip
FROM alb_logs
GROUP BY request_verb, client_ip
LIMIT 100;
```

Another query shows the URLs visited by Safari browser users:

```
SELECT request_url
FROM alb_logs
WHERE user_agent LIKE '%Safari%'
LIMIT 10;
```

The following example shows how to parse the logs by `datetime`:

```
SELECT client_ip, sum(received_bytes)
FROM alb_logs_config_us
WHERE parse_datetime(time,'yyyy-MM-DD''T''HH:mm:ss.SSSSSS''Z')
  BETWEEN 
    parse_datetime('2018-05-30-12:00:00','yyyy-MM-DD-HH:mm:ss')
    AND
    parse_datetime('2018-05-31-00:00:00','yyyy-MM-DD-HH:mm:ss')
GROUP BY client_ip;
```