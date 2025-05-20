# Daily-SPL-Basics
Some Splunk queries that might be useful for SOC routines

All of these queries are just "**simple**" usecases to give you "**ideas**" and key concepts that could help you in your SOC environment

--------------------------------------------------------------------------------------------------------------------------------------------------------
### Incoming Traffic from Internet
#### Incoming allowed src_ip
```
index="your index name" eventtype=traffic action!=blocked 
NOT src_ip IN ("192.168.0.0/16", "172.16.0.0/12", "10.0.0.0/8")
| timechart count by src_ip useother=false
```
#### Incoming allowed src_port
```
index="your index name" eventtype=traffic action!=blocked 
NOT src_ip IN ("192.168.0.0/16", "172.16.0.0/12", "10.0.0.0/8")
| timechart count by src_port useother=false
```

--------------------------------------------------------------------------------------------------------------------------------------------------------
### Outgoing Traffic from Inside
#### Outgoing allowed src_ip
```
index="your index name" eventtype=traffic action!=blocked 
src_ip IN ("192.168.0.0/16", "172.16.0.0/12", "10.0.0.0/8")
NOT dest_ip IN ("192.168.0.0/16", "172.16.0.0/12", "10.0.0.0/8")
| timechart count by src_ip useother=false
```
#### Outgoing allowed src_port
```
index="your index name" eventtype=traffic action!=blocked 
src_ip IN ("192.168.0.0/16", "172.16.0.0/12", "10.0.0.0/8")
NOT dest_ip IN ("192.168.0.0/16", "172.16.0.0/12", "10.0.0.0/8")
| timechart count by src_port useother=false
```

For all of these kinds of queries, there are many more aspects to monitor like `action=blocked`, `protocol=udp`, `src_ip!=<internal IP ranges>` and etc.


The main goal of these charts is to monitor the baseline traffic of your company and detect any abnormal traffic activity.

--------------------------------------------------------------------------------------------------------------------------------------------------------
### Important Services like DNS or HTTP

to avoid any Denial of Service or huge data transfer through a specific app or user

```
index="your index name" sourcetype=traffic
service IN (HTTP, HTTPS) 
| timechart count by src_ip useother=false
```
```
index="your index name" sourcetype=traffic service IN (HTTP, HTTPS) 
| stats sum(bytes_in) as rcv, sum(bytes_out) as sent by src_ip
```

--------------------------------------------------------------------------------------------------------------------------------------------------------
### Scanning
#### Port Scan
```
index=* (tag=network AND tag=communicate) earliest=-1h
| stats dc(dest_port) as num_dest_port, dc(dest_ip) as num_dest_ip by src_ip
| where num_dest_port > 50 OR num_dest_ip > 50
```
#### Ping Scan
```
index="your_index" sourcetype="your_sourcetype" icmp_type=8 
| timechart span=1m count by src_ip 
| where count > 10
```
there is also other scans like SYN, ARP, SV and ... scans.

--------------------------------------------------------------------------------------------------------------------------------------------------------
### Windows BruteForce Attack
to detect any foolish try
#### By Number of Attempts
```
index="your windows index name" host IN ("your DC") EventCode=4625 
| stats count  as attempts by src
| where attempts > 2
```
#### By Number of Dest Users
```
index="your windows index name" host=* EventCode=4625
| stats dc(user) as Users by src
| where Users>1
| sort - Users
```
#### By All Values
```
index="your windows index name" EventCode=4625
| stats dc(user) as UserCount,  values(user) count as AttemptCount by src
| where UserCount > 2 AND AttemptCount > 1
| sort - AttemptCount
```

--------------------------------------------------------------------------------------------------------------------------------------------------------

### File Transfer 
#### Send & Receive All Network
```
index="your index name" sourcetype="traffic" 
| eval sentbyte=(sentbyte)/(1024*1024)
| eval rcvdbyte=(rcvdbyte)/(1024*1024)
| stats sum(sentbyte),sum(rcvdbyte) by dest_ip,src_ip
| addcoltotals
```

#### Send & Receive From Inside to the Internet
```
index="your index name" eventtype=traffic src_ip IN (192.168.0.0/16,172.16.0.0/16,0.0.0.0,172.22.0.0/16) 
NOT dest_ip IN (192.168.0.0/16,172.16.0.0/16,172.28.0.0/16,0.0.0.0,172.22.0.0/16)
| stats count AS event_count sum(bytes_in) AS bytes_in sum(bytes_out) AS bytes_out sum(bytes) AS bytes_total by src
| eval mb_in=round(bytes_in/1024/1024)
| eval mb_out=round(bytes_out/1024/1024)
| eval mb_total=round(bytes_total/1024/1024)
| table src,mb_in,mb_out,mb_total 
| sort - mb_total
```

#### Send & Receive by User
```
index="your index name" user=*
| stats count AS event_count sum(bytes_in) AS bytes_in sum(bytes_out) AS bytes_out sum(bytes) AS bytes_total by user
| eval mb_in=round(bytes_in/1024/1024)
| eval mb_out=round(bytes_out/1024/1024)
| eval mb_total=round(bytes_total/1024/1024)
| sort - mb_total
| head 10
```
--------------------------------------------------------------------------------------------------------------------------------------------------------

### HTTP Status Codes

#### 1xx , 2xx , 3xx , 4xx , 5xx
```
index="your index name" sourcetype=traffic
|rename http_retcode as status_code 
| stats count(eval(status_code<100)) as under100,
    count(eval(status_code>=100 AND status_code<200)) as ret100,
    count(eval(status_code>=200 AND status_code<300)) as ret200,
    count(eval(status_code>=300 AND status_code<400)) as ret300,
    count(eval(status_code>=400 AND status_code<500)) as ret400,
    count(eval(status_code>=500)) as ret500,  count as total_count by http_host
| sort -total_count
```


some HTTP Response codes must be monitores in more details like 301 (Redirect) or 429 (Too Many Request) :

#### HTTP Status Code 301
```
index="your index name" http_method=get  http_retcode=301 NOT http_url="/"
|stats count values(http_url) by src
|sort - count
```

