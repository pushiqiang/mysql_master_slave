# mysql_master_slave
##mysql master-slave configuration; mysql 主从配置详解（超简单）

Mysql内建的复制功能是构建大型，高性能应用程序的基础。将Mysql的数据分布到多个系统上去，这种分布的机制，是通过将Mysql的某一台主机的数据复制到其它主机（slaves）上，并重新执行一遍来实现的。

复制过程中一个服务器充当主服务器，而一个或多个其它服务器充当从服务器。
主服务器将更新写入二进制日志文件，并维护文件的一个索引以跟踪日志循环。这些日志可以记录发送到从服务器的更新。当一个从服务器连接主服务器时，它通知主服务器从服务器在日志中读取的最后一次成功更新的位置。
从服务器接收从那时起发生的任何更新，然后封锁并等待主服务器通知新的更新。MySQL复制基于主服务器在二进制日志中跟踪所有对数据库的更改(更新、删除等等)。<br>
因此，要进行复制，必须在主服务器上启用二进制日志。<br>

主从服务器利用MySQL的二进制日志文件，实现数据同步。二进制日志由主服务器产生，从服务器响应获取同步数据库。

你的任何改变主服务器数据库的操作，都会同步到从服务器上。请注意当你进行复制时，所有对复制中的表的更新必须在主服务器上进行。否则，你必须要小心，以避免用户对主服务器上的表进行的更新与对从服务器上的表所进行的更新之间的冲突。即单向更新  master==>slave 。<br>


设置mysql主从配置的优点：
-----
  解决web应用系统，数据库出现的性能瓶颈，采用数据库集群的方式来实现查询负载；一个系统中数据库的查询操作比更新操作要多得多，通过多台查询服务器将数据库的查询分担到不同的查询服务器上从而提高查询效率。<br>
  Mysql数据库支持数据库的主从复制功能，使用主服务器进行数据的插入、删除与更新操作，而从服务器则专门用来进行数据查询操作，这样可以将更新操作和 查询操作分担到不同的数据库上，从而提高了查询效率。

MySql复制的基本过程：
-----

  　　1　Slave 上面的IO线程连接上 Master，并请求从指定日志文件的指定位置(或者从最开始的日志)之后的日志内容;<br>
  　　2　Master 接收到来自 Slave 的 IO 线程的请求后，通过负责复制的 IO线程根据请求信息读取指定日志指定位置之后的日志信息，返回给 Slave 端的 IO线程。返回信息中除了日志所包含的信息之外，还包括本次返回的信息在 Master 端的 Binary Log 文件的名称以及在 BinaryLog 中的位置;<br>
  　　3　Slave 的 IO 线程接收到信息后，将接收到的日志内容依次写入到Slave端的RelayLog文件(mysql-relay-lin.xxxxxx)的最末端，并将读取到的Master端的bin-log的文件名和位置记录到 master-info文件中，以便在下一次读取的时候能够清楚的高速Master“我需要从某个bin-log的哪个位置开始往后的日志内容，请发给 我”<br>
  　　4　Slave 的 SQL 线程检测到 Relay Log 中新增加了内容后，会马上解析该 Log 文件中的内容成为在 Master端真实执行时候的那些可执行的 Query 语句，并在自身执行这些 Query。这样，实际上就是在 Master 端和 Slave端执行了同样的 Query，所以两端的数据是完全一样的。<br>


我在win7和ubuntu12上面做的测试，版本都是mysql-5.5.46，结果表明跨操作系统也是可以完成数据库主从配置的；主服务器（ubuntu笔记本），从服务器就（win7台式）;从服务器版本一定要高于等于主服务器的版本。
----------------

###主服务器(ubuntu)配置

1修改主服务器master:
```
   #vi /etc/my.cnf       也可能在/etc/mysql/my.cnf
       [mysqld]
       log-bin=mysql-bin   //[必须]启用二进制日志
       server-id=222      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```
2重启mysql	service mysql restart

3在主服务器上建立帐户并授权slave:
   ```
    grant replication slave on *.* to 'pushiqiang'@'%' identified by 'q123';
    //一般不用root帐号，%为通配符，表示所有客户端都可能连，只要帐号，密码正确，此处可用具体客户端IP代替，如192.168.1.100，加强

安全。
```

4登录主服务器的mysql，查询master的状态
```
   mysql>show master status;
   +------------------+----------+--------------+------------------+
   | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
   +------------------+----------+--------------+------------------+
   | mysql-bin.000001 |      107 |              |                  |
   +------------------+----------+--------------+------------------+
```
记录下File域的值（mysql-bin.000001）Position域的值（107）后面从服务器配置需要用到

###从服务器（win7）配置

1修改从服务器slave:
```
   #vi /etc/my.cnf       win7下为 C:\Program Files (x86)\MySQL\MySQL Server 5.5\my.ini
       [mysqld]
       #log-bin=mysql-bin   //启用二进制日志
       server-id=222      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```
2重启mysql	service mysql restart	
		win7下 net stop/start mysql

3配置从服务器Slave：
```
   mysql>change master to master_host='192.168.1.106',master_user='pushiqiang',master_password='q123',
         master_log_file='mysql-bin.000001',master_log_pos=107;   //注意不要断开，308数字前后无单引号。
```
master_host:主服务器的ip；
master_port:端口号；（如果默认3306的话不需要指定）；
mstart_user:登陆主服务器mysql用户；
master_password:登陆主服务器mysql密码；
master_log_pos:从主服务器复制文件的第几个位置进行复制；即前面记录的master的状态中的File域的值
master_log_file:主服务器中的数据库复制文件；即前面记录的master的状态中的Position域的值

4启动从服务器复制功能   
```
   mysql>start slave;    
```
5检查从服务器复制功能状态：
```
   mysql> show slave status\G；
	      Slave_IO_Running: Yes    //此状态必须YES
        Slave_SQL_Running: Yes     //此状态必须YES
```
	若Slave_IO_Running: Connecting 则查看主服务器的3306端口是否允许远程访问，用telnent master 3306 测试

###允许3306远程访问
查看3306时候允许远程访问，若只有127.0.0.1:3306 则只允许本地
netstat -an|grep 3306

将/etc/mysql/my.cnf文件中	bind-address =127.0.0.1注销
<br>
现在你对主服务器的任何更新操作都将同步到从服务器，你可以试试建表，插入数据，删除数据，看是否同步成功(从服务器的更新不会同步到主服务器)。
