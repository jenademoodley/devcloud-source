+++
date = 2020-07-17
draft = false
title = 'Improving EFS Performance'
url = 'improving-efs-performance-part-2'
description = 'How to diagnose EFS performance issues'
weight = 10
showAuthor = false
authors = ["jenademoodley"]
showHero = true
heroStyle = 'background'
categories = ['AWS']
tags = ['EFS']
series = ['EFS Performance']
series_order = 2

+++

As discussed in the part 1 of this blog post, the latency which occurs with EFS is per-file operation latency due to the nature of the NFS protocol, specifically with handling metadata. In order to improve performance of your application, you need to understand the nature of your application, i.e. is your application performing mainly read or write operations. There are a few ways to determine this, which is discussed in the NFS troubleshooting blog post, however an easy way to determine the activity of your EFS on your EC2 instance is to use the `mountstats` command.

```bash
sudo mountstats /mnt/efs
```

Output:
```
Stats for 127.0.0.1:/ mounted on /mnt/efs:
...
NFS byte counts:
  applications read 404148 bytes via read(2)
  applications wrote 1966484148 bytes via write(2)
  applications read 0 bytes via O_DIRECT read(2)
  applications wrote 0 bytes via O_DIRECT write(2)
  client read 0 bytes via NFS READ
  client wrote 1966484148 bytes via NFS WRITE

RPC statistics:
  599130 RPC requests sent, 599130 RPC replies received (0 XIDs not found)
  average backlog queue length: 0

GETATTR:
        126739 ops (21%)
        avg bytes sent per op: 313      avg bytes received per op: 240
        backlog wait: 0.005097  RTT: 1.268323   total execute time: 1.311964 (milliseconds)
SETATTR:
        122786 ops (20%)
        avg bytes sent per op: 360      avg bytes received per op: 260
        backlog wait: 0.004838  RTT: 4.192335   total execute time: 4.237853 (milliseconds)
LOOKUP:
        115717 ops (19%)
        avg bytes sent per op: 364      avg bytes received per op: 268
        backlog wait: 0.023964  RTT: 2.415324   total execute time: 2.497403 (milliseconds)
ACCESS:
        72755 ops (12%)
        avg bytes sent per op: 320      avg bytes received per op: 168
        backlog wait: 0.008604  RTT: 1.881768   total execute time: 1.959151 (milliseconds)
REMOVE:
        40000 ops (6%)
        avg bytes sent per op: 346      avg bytes received per op: 116
        backlog wait: 0.003750  RTT: 7.256800   total execute time: 7.280375 (milliseconds)
OPEN:
        30225 ops (5%)
        avg bytes sent per op: 455      avg bytes received per op: 441
        backlog wait: 0.016973  RTT: 5.637519   total execute time: 5.663623 (milliseconds)
CLOSE:
        30006 ops (5%)
        avg bytes sent per op: 336      avg bytes received per op: 176
        backlog wait: 0.010231  RTT: 0.626241   total execute time: 0.643738 (milliseconds)
WRITE:
        30002 ops (5%)
        avg bytes sent per op: 65893    avg bytes received per op: 176
        backlog wait: 0.012932  RTT: 18.157989  total execute time: 18.181355 (milliseconds)
RENAME:
        30000 ops (5%)
        avg bytes sent per op: 528      avg bytes received per op: 152
        backlog wait: 0.007867  RTT: 5.331867   total execute time: 5.347367 (milliseconds)
READDIR:
        867 ops (0%)
        avg bytes sent per op: 342      avg bytes received per op: 32059
        backlog wait: 0.003460  RTT: 6.455594   total execute time: 6.489043 (milliseconds)
OPEN_NOATTR:
        5 ops (0%)
        avg bytes sent per op: 389      avg bytes received per op: 275
        backlog wait: 0.000000  RTT: 2.800000   total execute time: 2.800000 (milliseconds)
SERVER_CAPS:
        3 ops (0%)
        avg bytes sent per op: 316      avg bytes received per op: 164
        backlog wait: 0.000000  RTT: 0.333333   total execute time: 0.333333 (milliseconds)
FSINFO:
        2 ops (0%)
        avg bytes sent per op: 312      avg bytes received per op: 152
        backlog wait: 0.000000  RTT: 0.500000   total execute time: 0.500000 (milliseconds)
LOOKUP_ROOT:
        1 ops (0%)
        avg bytes sent per op: 184      avg bytes received per op: 380
        backlog wait: 0.000000  RTT: 0.000000   total execute time: 0.000000 (milliseconds)
PATHCONF:
        1 ops (0%)
        avg bytes sent per op: 308      avg bytes received per op: 116
        backlog wait: 0.000000  RTT: 0.000000   total execute time: 0.000000 (milliseconds)
SECINFO_NO_NAME:
        1 ops (0%)
        avg bytes sent per op: 172      avg bytes received per op: 104
        backlog wait: 0.000000  RTT: 5.000000   total execute time: 6.000000 (milliseconds)
```

As you can see, my EFS is performing write heavy operations, and you can even see the RPC calls made most often to the EFS, the majority of which is `GETATTR` and `SETATTR`. These are for setting metadata attributes, indicating that my workload is also metadata based.

Depending on the type or workload you have, i.e. read intensive or write intensive workloads, there are different options you can configure in order to enhance the performance of your application using EFS.

## Improving read operation performance

As we discussed, latency experienced on EFS is due to metadata access sequence of the NFS protocol. We cannot change how files are accessed from the EFS, however, we know that the local disk used is faster than EFS. Therefore, if we want to improve access time for files, we can cache the files on the local disk, effectively reducing latency by taking the NFS protocol out of the equation. This can be done using a file system cache (or fs-cache), which was created for this very purpose. A popular file system cache we can use is `cachefilesd`. To show the effects of using a cache, we can perform a test in which we access a file before and after using the cache to observe the performance improvements.

### Setting up cachefilesd

We first need to install cachefilesd using the standard package manager of your distribution.

#### Amazon Linux, CentOS, RHEL

```bash
sudo yum install cachefilesd -y
```

#### Ubuntu
```bash
sudo apt-get update
sudo apt-get install cachefilesd
```

After installing the package, we need to start the `cachefilesd` daemon.

#### Systemd

```bash
sudo systemctl start cachefilesd
```

#### Upstart
```bash
sudo service cachefilesd start
```

We then need to instruct the NFS client to use the cache when accessing the EFS by specifying the `fsc` mount option when mounting the EFS.

#### Using the EFS Mount Helper

```bash
sudo mount -t efs -o fsc <EFS_ID> <MOUNT_POINT>
```

#### Using the NFS client

```bash
sudo mount -t nfs4 -o fsc,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <EFS_DNS>:/ <MOUNT_POINT>
```

### Testing the cache

To test the cache, we first need to create a small file on the EFS which we will be accessing.

```bash
dd if=/dev/urandom of=/mnt/efs/cache_file bs=4k count=128
```

To ensure that the cache is not in effect, we can first clear the pagecache, dentries and inodes using `/proc/sys/vm/drop_caches`

```bash
sudo echo 3 | sudo tee /proc/sys/vm/drop_caches && sudo sync
```

We can then test an uncached access to the file we initially created.

```bash
time dd if=/mnt/efs/cache_file of=$HOME/local_file bs=4k count=128
```

Output:
```
128+0 records in
128+0 records out
524288 bytes (524 kB) copied, 0.0176215 s, 29.8 MB/s

real    0m0.027s
user    0m0.002s
sys     0m0.000s
```

First time accessing the file from the EFS took 0.027 seconds, at a speed of 29.8 MB/s. If we access this file again, this file will be served from the cache.

```bash
time dd if=/mnt/efs/cache_file of=$HOME/local_file bs=4k count=128
```

Output:
```
128+0 records in
128+0 records out
524288 bytes (524 kB) copied, 0.00155109 s, 338 MB/s

real    0m0.009s
user    0m0.002s
sys     0m0.000s
```

The time taken to access the file was 0.009 seconds at a speed of 338 MB/s. This is 3 times faster than the initial access time. While the access time may seem small, as the latency occurs per-file operation, this time will drastically increase for larger number of files in deeper directories.

### Limitations and Caveats

By definition, the operating system will check the cache first before accessing the EFS. This means that, should a file not exist, the entire cache will be checked before accessing the EFS. This can induce performance issues if the cache is very large and new files are being accessed more frequently than older files. This can be mitigated by configuring more options on cachefilesd such as cache culling, which involves discarding objects from the cache that have been used less recently than other objects. This is explained further in the [`cachefilesd` man pages](https://linux.die.net/man/5/cachefilesd.conf).

## Improving write operation performance

While using `cachefilesd` can speed up access to an existing file, it does not help much for write operations as the file may not yet exist (in the case of new files), and the NFS protocol would still need to access the metadata in order to perform updates such as updating the last modified time, and update the block size. In order to get around this limitation, we can take advantage of the resilient nature of EFS. 

EFS is designed to be accessed by thousands of clients simultaneously, and emphasis is placed on throughput of data written to the file system from all clients. This means that EFS is more situated for parallel workloads, or rather it is with parallel workloads that EFS really shines. As such, we can improve write performance by performing our commands in parallel. This can be done using two methods:

1. Use multiple clients (i.e. use multiple EC2 instances). 
2. Use a tool such as GNU parallel or fpsync to run the commands in parallel.

The first option is the more recommended option as it may be difficult to rebuild your application to execute operations in parallel. However, regular commands run on multiple files (for example `cp`, `mv`, or `rsync`) would benefit greatly if configured to operate on files in parallel. By default, these commands perform serial operations, i.e. it would perform the action on one file at a time. Performing the operations on multiple files simultaneously with a single command would greatly improve the performance. To demonstrate the results, as well as the ease of using a parallel utility, we can perform the below test using GNU parallel. 

### Installing GNU Parallel

The commands to install GNU parallel would differ per distribution. For RPM-based distributions (RedHat, CentOS, Amazon Linux), it is available in the `epel` repository. For Ubuntu, it is available in the default repository. The commands to install GNU parallel would be as below:

#### RHEL 7
```bash
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum install parallel -y
```

#### Amazon Linux, RHEL 6
```bash
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
sudo yum install parallel -y
```

#### Amazon Linux 2
```bash
sudo amazon-linux-extras install epel
sudo yum install parallel -y
```

#### CentOS 6, CentOS 7
```bash
sudo yum install epel-release
sudo yum install parallel -y
```

#### Ubuntu
```bash
sudo apt-get update
sudo apt-get install parallel
```

### Running commands in parallel

We first need to create a large number of small files on our local disk. In my test, I have created 10000 files. For reference, I am using a standard `t2.large` EC2 instance running Amazon Linux 2, with a general purpose (gp2) volume of 8 GB in size.

```bash
mkdir /tmp/efs
for each in $(seq 1 10000);
  do SUFFIX=$(mktemp -u _XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX);
  sudo dd if=/dev/zero of=/tmp/efs/${SUFFIX} bs=64k count=1; 
done
```

We can then copy these files to the EFS with `rsync` using the standard serial method, and measure the time it takes to copy the files.

```bash
cd /tmp/efs; time sudo find -L . -maxdepth 1 -type f -exec rsync -avR '{}' /mnt/efs/ \;
```

Output:
```
real    9m11.494s
user    0m34.186s
sys     0m10.303s
```

The results show that it took longer than 9 minutes to copy 10000 files to the EFS using `rsync`. If we purge the directory by [dropping the cache as seen above](#testing-the-cache), and use `rsync` in parallel:
```bash
cd /tmp/efs; time find -L . -maxdepth 1 -type f | sudo parallel rsync -avR {} /mnt/efs/
```
Output
```
real    5m41.274s
user    1m1.279s
sys     0m37.556s
```

The files took longer than 5 minutes, effectively reducing the copy time by almost half. Furthermore, GNU parallel runs as many parallel jobs as there are CPU cores. As the `t2.large` instance contains 2 cores, this explains why the time was almost halved. However, GNU parallel allows you to specify the number of jobs by using the `--jobs` flag. If we set the `--jobs` flag to 0, it would try to run as many jobs as possible with the amount of CPU available.

```bash
cd /tmp/efs; time find -L . -maxdepth 1 -type f | sudo parallel --jobs 0 rsync -avR {} /mnt/efs/
```

Output
```
real    3m25.578s
user    1m2.320s
sys     0m45.163s
```

Using as many jobs as possible, we reduced the time taken to about a third of the time it would have taken when using the serial copy method. You can also edit and configure GNU parallel commands as your needs. You can find a more detailed tutorial of GNU parallel [here](https://www.gnu.org/software/parallel/parallel_tutorial.html#GNU-Parallel-Tutorial).

## Conclusion

Knowing your application's usage patterns can greatly assist you with configuring your EFS. Using the above suggestions correctly can help you to use the EFS to it's full potential, however incorrect use of these suggestions, such as implementing `cachefilesd` for an application which only accesses recent data on the EFS, can cause further performance degradation. There are additional caching options you can look at such as [PHP-opcache](https://www.php.net/manual/en/book.opcache.php) for PHP-based websites, and incorporating a CDN such as [CloudFront](https://aws.amazon.com/cloudfront/), however these are website specific caching mechanisms. You need to identify a good mix of services and caching mechanisms in order to improve your application performance on EFS, and I hope this blog post gives you a good starting point on improving your experience with EFS.