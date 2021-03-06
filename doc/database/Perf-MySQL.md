This is the copy of [original blog](http://www.critiqueslibres.com/blog/?p=1603)

MySQL slow performance with updates
----

Publié le 30 août 2012 par Pierre

This week-end I moved the site to a new server at my hoster (www.ovh.net). After five years, it was a serious hardware upgrade, going from 2G of RAM to 16G of RAM for example and from a pentium dual core to a i3 quad core machine. But I was very disappointed by the performance of the new server and by looking in the slow query log I could see that MySQL was the culprit.

By configuration, it is possible to tell MySQL to log all SQL taking more than x second (or microsecond), which is very useful to diagnose performance. So in the `/etc/my.cnf` file, I added this

```
slow_query_log=1
long_query_time=1
slow-query-log-file=/data2/logs/slow_queries.log
```

I could immediately see that quite regularly, a very simple update statement (used to increment a counter based on a primary key) was taking more than 1 second, which pointed to a disk issue. It was also obvious that the issue was related to write, because the log was full of update/insert statements. The new server has two disks mirrored at the software level (RAID SOFT). I started to search for disks performance issue with MySQL, and decided to test with various documented parameters.

To do so, I did a small benchmark. I created a  file with 5000 updates, executed it with mysql while spooling to a file, then I parsed the generated file to extract the execution time of each update.

Concretly, I created a file called `testperf.sql` containing


```
\T perf.log
update test_table set col1 = col1+1,col2=col2+1 where id=29890;
update test_table set col1 = col1-1,col2=col2-11 where id=29890;
etc... 5000 times
quit
```

I started mysql and sourced the sql fie

```sh
$mysql -u <user> -p <password> <dbname>
mysql>source testperf.sql
mysql>quit
```

then in the shell, I parsed the generated `perf.log` file, extracting each execution time and making the sum

```sh
$ grep "Query OK" perf.log | sed -e "s/.*(\(.*\)sec)/\1/" | sort -n > t.log
$ ( echo 0 2k ; sed 's/$/ +/' t.log ; echo p ) | dc
```

The result was stunning.

While on the old server I had something like 170 seconds, on the new server it was more than the double.

First I tried to change the parameter `innodb_flush_method`. On the old server, it made a very big difference;

```
with the default value: 170 seconds
with O_DSYNC: 128 seconds.
with O_DIRECT: 86 seconds
```

But on the new server, changing this parameter made the performance only worst.

Then I altered the table to be `MyISAM` instead of `InnoDB` (alter table test storage MyIsam) and re-run the performance test: the performance was now excellent. So I got the indication that the performance problem was caused by the transaction. And indeed, if I turned off the `autocommit` (set `autocommit = 0`), I got good performance with the InnoDB engine as well. Now it was clear that the problem was due to the writes to disk of the log buffer (which happens every time a transaction is committed), with the behavior that from time to time very simple updates were hanging for 1 or 2 seconds (probably waiting for the OS to complete the write).

Luckily there is a way to tell MySQL that it does not need to wait for the write to be complete every time there is a commit: innodb_flush_log_at_trx_commit. By setting this parameter to 0, you are not completly protected against data loss in case of an outage because the latest committed transactions might not have been written to disk. According to the documentation, MySQL flushes anyway the commited transactions to disk every second, so the risk of data loss is not very big: you could lose 1 second of transaction at worst (which is absolutely not an issue in my case).

With this parameter, the DML statements are flying, in fact the whole test runs in less than a second which is 1000 times faster!!

Below are the parameters related to performance in my ini file. I want to do more testing, because I am not completly satisfied with this option. I want to test the disks with fio, which looks like a nice tools to test disks performance. But in the meantime, my database is now flying and I am very happy about that!

```
key_buffer_size=1024M
innodb_buffer_pool_size=4096M
#This setting makes a huge performance difference on this system !!
innodb_flush_log_at_trx_commit=0
innodb_log_file_size=250M
innodb_log_buffer_size=16M
slow_query_log=1
long_query_time=0.5
slow-query-log-file=/data2/logs/slowqueries.log
general_log=0
general_log_file=/data2/logs/mysql_generalquery.log
```

Ce contenu a été publié dans Le coin technique par Pierre, et marqué avec MySQL performance. Mettez-le en favori avec son permalien.


