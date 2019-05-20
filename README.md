
# elasticall

Shell tool to query and monitor Elasticsearch

See the documentation [here](https://slac-lcls.github.io/elasticall/)

## Motivation

I would like to be able to give such kind of high level commands from the shell: 

1. "Computer, tell me how many log lines entered Eleasticsearch in the last hour."
2. "Computer, give me a random sample of log lines stored into Elasticsearch index lcslogs in the last month."
3. "Computer, tell me how long the Elasticsearch has been running and how much heap space is available".
4. "Computer, give me all log lines produced in the last hour containing the word 'error'."

Commands [1,2,3,4] provide interesting output but there is more we can get than the sheer answer. Running `time` on command [2] is the best way I can figure out to measure the **search time performance** of Elasticsearch. Indeed, to take a random sample Elasticsearch is forced to visit all the sample-space each time. If you repeat two time the same command you should get similar response time.

Looking at command [4] `elasticall` seems similar to a previously developed tool called [elastico](https://github.com/slac-lcls/elastico), which also runs Elasticsearch queries from the shell. *elastico* though is a tool customized to search into logs, that is, to search into our *lclslogs* index. Conversely, *elasticall* can run very general parametric queries and it is index agnostic. Respect to *elastico*, *elasticall* is more general, more powerfull and more difficult to use.


`elasticall` was made to throw very complex queries to Elasticsearch and beutify the output results. As such, it is a tool that frees the user from the tedious interactions with JSON. Taking random sample is just anohter, not trivial, Elasticsearch query.

`elasticall` calls always a **recipe script file**, the recipe file containes [1] the JSON command to send to Elasticsearch in a very readable format [2] preprocessing routines and [3] postprocessing routines. I give a few examples of script files below, using them as template it is not difficult to extend *elasticall* capabilities but adding new scripts. 

`$> elasticall -h` provides a brief manual page. The provided script files available in directory *query* and are commented.


## Installation & run

***CAVEAT.*** The procedure is tested working for **lclslogs** Elasticsearch index and for the program `elasticall` to be installed in *psmetric01*. Other users in other networks need to tweak some details, as the Elasticsearch URL, in the source code.

- Get the source from GitHub
- Run the installation script: `$> sudo install`
- Test run, shows the builtin manual: 
```
$> elasticall -h
```
```
=> MANUAL TEXT ...
```

- Easy task, show the number all log lines stored into LCLS-logs index.

```
$> elasticall --s0='*' /usr/local/etc/elasticall/paramSearchAndCount
```

```
=> querys: '*', num.doc: 557,668,598
```

## Heads up ! 

**CAVEAT for LCLS users.** Before you do any test you must be aware of this important fact. `elasticall` runs in *psmetric01* but the Elasticsearch server runs in *psmetric04*. If you run a search that takes time and kill `elasticall` before it ends, by e.g. `Ctrl+C,` or `kill`, the search will keep on running on *psmetric04*! 

If you intend to run multiple queries via `elasticall` and kill them librerally then keep an eye on *psmetric04* load with `top`. A CPU load of about *500%* in `top` is expected when you run long and complex queries.


## Random Sample to measure search performace

**Def.** What is called here **random sample** is a shortcut for 'random sample without replacement'. This is what I implemented.

A random sample, if well taken, implies that all the sample space must be visited at each run. It makes no sense to cache results. Also, if the sample space is large enough, it is impossible to cache it in RAM.

***CASE.1*** Random sample without parameters, by default runs on all of Elasticsearch *lclslogs* index and returns 20 lines. 

```
$> time elasticall queries/paramRandomSample2
time elasticall /usr/local/etc/elasticall/paramRandomSample2 
ATTENTION: this query can take some time ... 
total hits: 557,365,125
Mar  3 04:42:38 psnfs02 kernel: [4453638.338888] sd 1:0:45:0: [sdaj] Bad block number requested
Feb 27 23:48:26 psnfs02 kernel: [4176799.486066] sd 1:0:45:0: [sdaj] Bad block number requested
==== CUTTED LINES ====
Mar  9 02:49:47 psana1216 monit[1956]: Processing queued event /var/monit/1549399756_1c92a80

real    1m18.535s
user    0m0.191s
sys     0m0.026s
```

**OBSERVE**
- There are about 550 milion lines stored into Elasticsearch *lclslogs* index. 
- The search took 1 minute and 20 seconds, it is quite a lot. This time is expected to increase on increasing the amount of data stored into Elasticsearch.


***CASE.2*** Random sample of 5 lines over all lines produced in between `now` and 3 days ago `now-3d`.

```
 time elasticall --s0=5 --s2='now-3d' --s3='now'  \
                  /usr/local/etc/elasticall/paramRandomSample2
```

```
ATTENTION: this query can take some time ... 
total hits: 10,214,480
May 14 01:07:16 psbackup05 kernel: sd 1:0:44:0: [sdar] Sense Key : Not Ready [current] [descriptor] 
May 13 17:24:41 ioc-hpl-03.pcdsn kernel: EtherCAT 0: Domain 0: 4 working counter changes - now 9/9.
May 16 00:50:40 psanagpu104 kernel: [2959316.460953] NFS: state manager: check lease failed on NFSv4 server psnfs02 with error 13
May 15 03:50:06 psrelay.pcdsn dhcpd: DHCPDISCOVER from 0c:c4:7a:1c:3c:2b via eth1.661: network 172.21.48/22: no free leases
May 14 04:50:40 psbackup05 kernel: mpt3sas_cm0: #011request_len(4096), underflow(4096), resid(4096)

real    0m1.557s
user    0m0.188s
sys     0m0.029s
```

***OBSERVE***. 
- "total hits: 10,214,480". This query ran across 10 milion documents to get its 5 elements selection.
- The query took about 0.2 seconds to be resolved.
- This amount of time is expected not to grow much in future.

***CASE.3*** Random sample of 5 lines over all lines produced in the last month and matching the Lucene query `nmingott AND (NOT psmetric*)`. 

```
time elasticall --s0=5 --s1='nmingott AND (NOT psmetric*)' \
                   /usr/local/etc/elasticall/paramRandomSample2 
```

```
ATTENTION: this query can take some time ... 
total hits: 101,995
Dec 26 07:55:01 psanagpu101 systemd[1]: Starting Session 1819742 of user nmingott.
Dec 16 12:00:01 psanagpu101 systemd[1]: Started Session 1665130 of user nmingott.
Dec 31 22:45:01 psanagpu101 CROND[2202]: (nmingott) CMD (export CRON_RUN="true"; /reg/neh/home/nmingott/cronScripts/thisMachineStatusToElastic.rb)
Dec 14 10:10:01 psanagpu103 CROND[9996]: (nmingott) CMD (export CRON_RUN="true"; /reg/neh/home/nmingott/cronScripts/thisMachineStatusToElastic.rb)
Dec 20 06:40:01 psana102 systemd: Starting Session 5116 of user nmingott.

real    0m0.455s
user    0m0.205s
sys     0m0.023s
```

## Random Sample to detect anomalies 

The validity of the method proposed in this section relies on **one powerfull assumtion**: *when something starts to go wrong in a machine there will be a massive production of similar error messages*.

Sometimes the assumption may not hold, e.g. a machine may have the powersupply broken, there will not be any error log in this case, the machine will turn off. We disregard this kind of failures now.

The supposedly there is something wrong in our machines and a consequent massive production of error messages how could we spot the fact without too much technicalities? We just take a random sample, if we see repeated messages there is a good probability we have a problem.

Then, there are problem that are logstanding, we can not fix them in one day, so it is often usefull to compare our current random sample with an old one. 

Let's see an example. We start by taking a random sample of messages from the few last days. 

```
$> time elasticall --s0=20 --s2='now-2d' --s3='now'\
                    /usr/local/etc/elasticall/paramRandomSample2
```
```
ATTENTION: this query can take some time ... 
total hits: 6,876,159
May 16 11:10:01 psana1408 crond[30014]: (root) FAILED to authorize user with PAM (Failure setting user credentials)
May 16 06:25:11 psanagpu104 rpc.gssd[1273]: ERROR: No credentials found for connection to server psnfs01.pcdsn
May 14 20:43:38 psanagpu104 rpc.gssd[1273]: ERROR: gssd_refresh_krb5_machine_credential: Input/output error while initializing krb5 context
May 16 02:41:44 psrelay.pcdsn dhcpd: DHCPDISCOVER from 0c:c4:7a:1d:0d:e7 via eth1.661: network 172.21.48/22: no free leases
May 16 17:33:23 psanagpu104 rpc.gssd[1273]: ERROR: gssd_refresh_krb5_machine_credential: Input/output error while initializing krb5 context
May 15 05:47:00 ioc-hpl-03.pcdsn kernel: EtherCAT 0: Domain 0: 4 working counter changes - now 9/9.
May 15 03:41:35 psanagpu104 rpc.gssd[1273]: ERROR: gssd_refresh_krb5_machine_credential: Input/output error while initializing krb5 context
May 16 10:28:18 ioc-cxi-usrmot1.pcdsn kernel: input: Winbond Electronics Corp Hermon USB hidmouse Device as /class/input/input3
May 15 18:11:21 ioc-hpl-03.pcdsn kernel: EtherCAT WARNING: Datagram ffff810364a3b198 (domain0-0-main) was SKIPPED 2 times.
May 14 20:51:03 psbackup05 kernel: sd 1:0:129:0: [sddt] CDB: Read(16) 88 00 00 00 00 00 00 00 00 00 00 00 00 08 00 00
May 16 02:49:46 psbackup05 kernel: mpt3sas_cm0: #011handle(0x003a), ioc_status(scsi data underrun)(0x0045), smid(361)
May 14 23:52:04 psanaphi102 systemd-logind[1486]: Removed session 233810.
May 14 18:21:45 ioc-hpl-03.pcdsn kernel: EtherCAT WARNING 0: 2 datagrams UNMATCHED!
May 16 06:55:31 psanaphi101 systemd[1]: Created slice user-15622.slice.
May 15 17:05:10 psmetric04 telegraf: 2019-05-16T00:05:10Z E! [inputs.snmp]: Error in plugin: agent switch-tst-b950-208: performing get on field hostname: Request timeout (after 3 retries)
May 15 08:17:04 psanagpu104 rpc.gssd[1273]: ERROR: gssd_refresh_krb5_machine_credential: Input/output error while initializing krb5 context
May 15 02:32:46 psanagpu104 rpc.gssd[1273]: ERROR: No credentials found for connection to server psnfs02.pcdsn
May 16 05:36:11 psanagpu104 rpc.gssd[1273]: ERROR: gssd_refresh_krb5_machine_credential: Input/output error while initializing krb5 context
May 15 12:37:04 daq-mec-cam01 NetworkManager[1157]: <info>  [1557949024.8139] device (enp5s0f1): Activation: starting connection 'Wired connection 1' (623b2eaa-e903-30ca-af9b-26e4a0cb6b6e)
May 15 21:03:48 psanagpu104 rpc.gssd[1273]: ERROR: gssd_refresh_krb5_machine_credential: Input/output error while initializing krb5 context

real    0m0.837s
user    0m0.194s
sys     0m0.024s
```

We see there is something to fix in *psanagpu104* and in general it is the service *rpc.gssd*, it seems an authentication problem.

Let's compare with a sample of 20 messages produced between 1 and 2 months ago. 

```
 $> time elasticall --s0=20 --s2='now-2M' --s3='now-1M' \
                     /usr/local/etc/elasticall/paramRandomSample2
```

```
ATTENTION: this query can take some time ... 
total hits: 72,552,603
Mar 18 17:56:06 psana1119 monit[1859]: Processing queued event /var/monit/1550881320_1eac350
Apr 11 04:15:21 psanagpu104 sshd[7769]: Received disconnect from 172.21.49.203: 11: disconnected by user
Mar 17 22:46:46 psana1403 monit[1785]: Processing queued event /var/monit/1549389644_1511570
Apr  6 19:21:38 psrelay.pcdsn dhcpd: DHCPDISCOVER from 0c:c4:7a:1c:3c:2a via eth1.661: network 172.21.48/22: no free leases
Apr 11 22:36:27 psrelay.pcdsn dhcpd: DHCPACK on 172.21.68.220 to 00:25:90:ac:66:9a via 172.21.68.2
Mar 29 23:22:30 psanaphi102 systemd[1]: Starting user-15622.slice.
Mar 17 01:06:21 psana1104 monit[1873]: Processing queued event /var/monit/1551421314_2642150
Mar 18 13:17:15 psana1217 monit[1698]: Processing queued event /var/monit/1549986733_10afb60
Mar 19 19:14:47 psana1209 monit[1864]: Processing queued event /var/monit/1548493934_fe8830
Mar 30 00:31:43 psanaphi102 systemd[1]: Stopping user-15622.slice.
Mar 17 01:46:50 psana1306 monit[1863]: Processing queued event /var/monit/1549396185_718460
Apr  5 09:04:48 psanagpu101 sshd[14699]: pam_unix(sshd:session): session opened for user psjhub by (uid=0)
Apr 11 21:53:36 ioc-hpl-03.pcdsn kernel: EtherCAT WARNING 0: 1 datagram UNMATCHED!
Mar 20 19:08:31 psana1310 monit[1852]: Processing queued event /var/monit/1550864313_9b97b0
Mar 18 22:57:42 psana1218 monit[1784]: Processing queued event /var/monit/1551420561_1f71400
Mar 19 19:56:36 psana1215 monit[1904]: Processing queued event /var/monit/1550863982_f465b0
Mar 30 07:51:33 psrelay.pcdsn dhcpd: DHCPDISCOVER from 0c:c4:7a:01:da:a0 via 134.79.165.2: network 134.79.165/24: no free leases
Mar 19 15:35:22 psana1402 monit[1842]: Processing queued event /var/monit/1550080196_16748d0
Mar 18 18:10:40 psana1103 monit[1874]: Processing queued event /var/monit/1549394307_8662c0
Apr 16 14:04:25 psrelay.pcdsn dhcpd: DHCPDISCOVER from 00:25:90:02:ef:d4 via 172.21.62.2

real    0m9.999s
user    0m0.188s
sys     0m0.028s
```

We see the service *rpc.gssd* does not appear at all in the past. This must be a recent problem. 

Something persisting from the past instead is in machine *psrealay*, a *dhcpd* error message. It seems some computer is not getting an IP. 

The analysis is very crude but we can already establish with a good degree of confidence that we ***may have a problem in *psanagpu104****. 

## Counting log lines 

Let's see the script *paramSearchAndCount* whose function is to count the number of docuements in the index lclslogs. 

Let's count all logs in the last hour.

```
$> elasticall --s0='*' --s1='now-1h' --s2='now' \    
/usr/local/etc/elasticall/paramSearchAndCount 
```

```
queries: '*', num.doc: 145,763
```

Le't count all document in the last 24 hours matching the Lucene query "lustre AND (ana*)".

```
$> elasticall --s0='lustre AND (ana*)' --s1='now-1d' --s2='now' /usr/local/etc/elasticall/paramSearchAndCount 
```
```
querys: 'lustre AND (ana*)', num.doc: 144
```

## Getting the status of Elasticsearch 

Getting the Elasticserch status and it Java virtual machine statistics is just one query away. Unfortunately the result are extermely messy.

```
$> curl -XGET 'http://psmetric04:9200/_nodes/stats?pretty=true'
```
```
... .. .. .. ... ....
      "jvm" : {
        "timestamp" : 1558133036326,
        "uptime_in_millis" : 340242256,
        "mem" : {
          "heap_used_in_bytes" : 2649697608,
          "heap_used_percent" : 50,
          "heap_committed_in_bytes" : 5255331840,
          "heap_max_in_bytes" : 5255331840,
          "non_heap_used_in_bytes" : 175657304,
          "non_heap_committed_in_bytes" : 190705664,
          "pools" : {
            "young" : {
              "used_in_bytes" : 567601824,
              "max_in_bytes" : 907345920,
              "peak_used_in_bytes" : 907345920,
              "peak_max_in_bytes" : 907345920
            },
.. .. .. .. ...
```


Let's build 


