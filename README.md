# **Web Server Log Analysis with Apache Hive**

## **Project Overview**
This project focuses on analyzing web server logs using **Apache Hive**. The logs contain IP addresses, timestamps, URLs, HTTP status codes, and user agents. The analysis includes:

- Counting total web requests
- Analyzing HTTP status code distribution
- Identifying the most visited pages
- Determining the most common user agents
- Detecting suspicious IP addresses
- Observing traffic trends over time
- Implementing **partitioning by status code** to optimize query performance

---

## **1️⃣ Setup and Execution**

### **1.1 Copy Log File to Hadoop Container**
```bash
docker cp web_server_logs.csv resourcemanager:/opt/hadoop-2.7.4/share/hadoop/mapreduce/
docker exec -it resourcemanager /bin/bash
cd /opt/hadoop-2.7.4/share/hadoop/mapreduce/
```

### **1.2 Upload Data to HDFS**
```bash
hdfs dfs -mkdir /input
hdfs dfs -put web_server_logs.csv /input
hdfs dfs -ls /input/*
```

---

## **2️⃣ Create Hive Tables**

### **2.1 Create an External Table**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS web_logs_external (
    ip STRING,
    log_timestamp STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/input/';
```

### **2.2 Verify Table Data**
```sql
SELECT * FROM web_logs_external LIMIT 10;
```

### **2.3 Create a Partitioned Table**
```sql
CREATE TABLE IF NOT EXISTS web_logs_partitioned (
    ip STRING,
    log_timestamp STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

### **2.4 Load Data into Hive**
```sql
LOAD DATA INPATH '/input/web_server_logs.csv' INTO TABLE web_logs;
```

---

## **3️⃣ Queries for Web Log Analysis**

### **3.1 Count Total Web Requests**
```sql
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/total_web_requests'
SELECT CONCAT_WS(' ', 'Total Requests:', CAST(COUNT(*) AS STRING)) FROM web_logs_partitioned;
```

### **3.2 Analyze Status Codes**
```sql
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/analyze_status_codes'
SELECT CONCAT_WS(' ', CAST(status AS STRING), CAST(request_count AS STRING))
FROM (
    SELECT status, COUNT(*) AS request_count
    FROM web_logs_partitioned
    GROUP BY status
) AS subquery;
```

### **3.3 Identify Most Visited Pages**
```sql
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/top_visited_pages'
SELECT CONCAT_WS(' ', url, CAST(visit_count AS STRING))
FROM (
    SELECT url, COUNT(*) AS visit_count
    FROM web_logs_partitioned
    GROUP BY url
    ORDER BY visit_count DESC
    LIMIT 3
) AS subquery;
```

### **3.4 Traffic Source Analysis**
```sql
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/traffic_sources'
SELECT CONCAT_WS(' ', user_agent, CAST(agent_count AS STRING))
FROM (
    SELECT user_agent, COUNT(*) AS agent_count
    FROM web_logs_partitioned
    GROUP BY user_agent
    ORDER BY agent_count DESC
) AS subquery;
```

### **3.5 Detect Suspicious IPs (More than 3 Failed Requests)**
```sql
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/suspicious_ips'
SELECT CONCAT_WS(' ', ip, CAST(failed_requests AS STRING))
FROM (
    SELECT ip, COUNT(*) AS failed_requests
    FROM web_logs_partitioned
    WHERE status IN (404, 500)
    GROUP BY ip
    HAVING COUNT(*) > 3
    ORDER BY failed_requests DESC
) AS subquery;
```

### **3.6 Analyze Traffic Trends (Requests Per Minute)**
```sql
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/traffic_trends'
SELECT CONCAT_WS(' ', time_minute, CAST(request_count AS STRING))
FROM (
    SELECT SUBSTR(log_timestamp, 0, 16) AS time_minute, COUNT(*) AS request_count
    FROM web_logs_partitioned
    GROUP BY SUBSTR(log_timestamp, 0, 16)
    ORDER BY time_minute
) AS subquery;
```

### **3.7 Query Partitioned Data for Status Code 404**
```sql
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/status_404'
SELECT CONCAT_WS(' ', ip, log_timestamp, url, user_agent, CAST(status AS STRING))
FROM web_logs_partitioned WHERE status = 404;
```

---

## **4️⃣ Export and View Results**
After executing the queries, results will be stored in HDFS. View them using:

```bash
hdfs dfs -cat /output/web_logs_analysis/total_web_requests/000000_0
hdfs dfs -cat /output/web_logs_analysis/analyze_status_codes/000000_0
hdfs dfs -cat /output/web_logs_analysis/top_visited_pages/000000_0
hdfs dfs -cat /output/web_logs_analysis/traffic_sources/000000_0
hdfs dfs -cat /output/web_logs_analysis/suspicious_ips/000000_0
hdfs dfs -cat /output/web_logs_analysis/traffic_trends/000000_0
hdfs dfs -cat /output/web_logs_analysis/status_404/000000_0
```

---

## **5️⃣ Challenges Faced & Solutions**
### **5.1 Dynamic Partitioning Issue**
- **Error:** `Dynamic partition strict mode requires at least one static partition column.`
- **Solution:** Enable dynamic partitioning:
  ```sql
  SET hive.exec.dynamic.partition = true;
  SET hive.exec.dynamic.partition.mode = nonstrict;
  ```

### **5.2 COUNT(*) Not Allowed in ORDER BY**
- **Error:** `Not yet supported place for UDAF 'COUNT'`
- **Solution:** Used **subqueries** to perform aggregation first, then applied `ORDER BY`.

### **5.3 CONCAT_WS() Type Mismatch**
- **Error:** `Argument type mismatch 'COUNT'`
- **Solution:** Used `CAST(COUNT(*) AS STRING)` to convert `BIGINT` to `STRING`.

---

## **6️⃣ Conclusion**
This project successfully analyzes web server logs using Apache Hive with **partitioning by status code** for improved query performance. The queries are optimized for large-scale data processing, and the results are efficiently stored in HDFS for further analysis.

---
