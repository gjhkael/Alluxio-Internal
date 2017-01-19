#Alluxio bug and fix bug

###1.Hive on alluixo 权限问题
如果hive使用kerberos来保证数据的安全问题，那么hive on alluxio会出现各种权限方面的问题。不过这在1.4.0已经解决，详情请看：[Hive permission issue fix](https://github.com/Alluxio/alluxio/pull/4453)

###spark on yarn on alluxio权限问题
第一个问题解决后，这个问题也随之解决。

###2.alluxio数据安全问题
alluxio现在使用的sample方式来管理数据，用户可以设置alluxio.security.login.username来指定任何用户，从而来操作该用户的数据。这边后期有机会为alluxio
增加基于kerberos的安全认证，有兴趣的同学可以联系我，一起做。

###3.alluxio多副本问题
alluxio数据只会在内存中存在一份，这样会导致mapreduce on alluxio的时候，因为没有数据备份，所以locality不如 mr on hdfs。
请见这个[mailing list](https://groups.google.com/forum/?fromgroups=#!topic/alluxio-users/Jmz_DmVLVjU)
###4.alluxio基于文件夹的ttl设置
这个问题在1.5.0会发布，见[Add ttl directory function](https://github.com/Alluxio/alluxio/pull/4458)
###5.alluxio makeConsistency命令
这个问题已经提了PR见[Make consistency](https://github.com/Alluxio/alluxio/pull/4686)




