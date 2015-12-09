实时的分布式sphinx索引配置及使用方法总结
coreseek文档：http://sphinxsearch.com/wiki/doku.php?id=sphinx_manual_chinese#需要的工具
需要更改/usr/local/coreseek/var/data  下的目录权限


    安装开始：
     cd /data/softwore
     wget  http://www.coreseek.cn/uploads/csft/4.0/coreseek-4.1-beta.tar.gz(只安装中文分词mmseg3)
     tar zxvf coreseek-4.1-beta.tar.gz
     cd coreseek-4.1-beta.tar.gz/
     安装mmseg
     $ cd mmseg-3.2.14  //根据具体的版本而定
     $ ./bootstrap    #输出的warning信息可以忽略，如果出现error则需要解决
     $ ./configure --prefix=/usr/local/mmseg3
     $ make && make install
     $ cd ..
     生成字典
     因为用到中文分词，需要生成字典，去安装目录，比如我的是 /home/changyou/mmseg.3.0b3/data/
     mmseg -u unigram.txt 该命令执行后，将会产生一个名为unigram.txt.uni的文件，将该文件改名为uni.lib，完成词
     遇到的问题:
     error: cannot find input file: src/Makefile.in
     或者遇到其他类似error错误时...
     解决方案：
     依次执行下面的命令，我运行'aclocal'时又出现了错误，解决方案请看下文描述
     yum -y install libtool
     aclocal
     libtoolize --force
     automake --add-missing
     autoconf
     autoheader
     make clean
     #####安装sphinx
     $ cd csft-4.1
     $ sh buildconf.sh    #输出的warning信息可以忽略，如果出现error则需要解决
     $ ./configure --prefix=/usr/local/sphinx  --without-unixodbc --with-mmseg --with-mmseg-includes=/usr/local/mmseg3/include/mmseg/ --with-mmseg-libs=/usr/local/mmseg3/lib/ --with-mysql-includes=/usr/local/webserver/mysql/include/mysql --with-mysql-libs=/usr/local/webserver/mysql/lib/mysql  //此处的MySQL依赖库需要根据具体安装的MySQL路径有关/lib/mysql/为文件夹,有些系统没有mysql文件夹 直接指定到include或lib下即
    #####如果提示mysql问题，可以查看MySQL数据源安装说明
    $ make && make install
    $ cd ..
    #####测试mmseg分词，coreseek搜索（需要预先设置好字符集为zh_CN.UTF-8，确保正确显示中文）
    $ cd testpack
    $ cat var/test/test.xml    #此时应该正确显示中文
    $ /usr/local/mmseg3/bin/mmseg -d /usr/local/mmseg3/etc var/test/test.xml



      安装sphinx报错解决方法如下：
      打开sphinx-2.1.9.release下的configure
      可能遇到的问题：
      如果提示libtool: unrecognized option `--tag=CC' ，请查看libtool问题解决方案
      有的系统下可能出现：expected `;' before ‘CSphTokenizer_UTF8SpaceSeg’，
      或者出现：configure: WARNING: unrecognized options: --with-mmseg, --with-mmseg-includes, --with-mmseg-libs
      是因为你没有进行随后的sh buildconf.sh操作
      生成当前系统对应的编译配置文件
      需要使用以下指令：
      $ sh buildconf.sh
      ## Linux环境下，如遇到pthread问题，请先直接执行以下指令在进行configur：
      $ LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
      $ export LD_LIBRARY_PATH
      ########如果出现undefined reference to `libiconv'的类似错误，可以按照如下方法处理：
      ########方法一：（Linux使用）
      ######## 直接执行：export LIBS="-liconv"
      #######然后make clean，再次configure后，进行编译安装make && make install
      ######## 方法二：
      ######## 首先configure，然后vim src/makefile
      ######## 在其中搜索lexpat，在其后加上 -liconv
      ######## 修改后该行应该为：-lexpat -liconv -L/usr/local/lib
      ######## 然后再次make && make install
      ######## 方法三：
      ######## 首先configure，然后vim config/config.h
      ######## 在其中搜索USE_LIBICONV，将其后的1修改为0
      ######## 然后再次make && make install
      下面开始sphinx与mysql的配置
      创建sphinx统计表，在coreseek_test库中执行。
      CREATE TABLE sph_counter
      (
           counter_id INTEGER PRIMARY KEY NOT NULL,
           max_doc_id INTEGER NOT NULL
      );
     配置数据源文件


    vi /usr/local/sphinx/etc/sphinx.conf
    写入如下内容配置
    source main
    {
           type    = mysql
           sql_host = localhost ＃此处为数据库地址
           sql_user = root #用户名
           sql_pass = root #密码
           sql_db  = db #数据库名称
           sql_port = 3306
           sql_sock = /tmp/mysql.sock (不是必须)
           sql_query_pre   = SET NAMES utf8
           sql_query_pre = REPLACE INTO sph_counter_product SELECT 1,MAX(id) from job
           #数据库中读取数据的sql
           sql_query  = SELECT id,name,companyname,radians(lat) as lat,radians(lng) as lng,salary,education,jobtype,secondtype,UNIX_TIMESTAMP(ctime) As ctime,UNIX_TIMESTAMP(mtime) As mtime from job where id<=(SELECT max_doc_id FROM sph_counter_product WHERE counter_id=1)
           #支持全文索引
           sql_field_string = name
           #支持属性过滤
           sql_attr_string = companyname
           sql_attr_float = lat
           sql_attr_float = lng
           sql_attr_uint = salary
           #sql_attr_uint = id
           sql_attr_uint = education
           sql_attr_uint = jobtype
           sql_attr_uint = secondtype
           sql_attr_timestamp = ctime
           sql_attr_timestamp = mtime
           sql_query_info_pre      = SET NAMES utf8
           sql_query_info  = SELECT id,name,companyname,radians(lat) as lat,radians(lng) as lng,salary,education,jobtype,secondtype,UNIX_TIMESTAMP(ctime) As ctime,UNIX_TIMESTAMP(mtime) As mtime FROM job order by ctime desc
    }
    source delta : main
    {
       sql_query_pre           = SET NAMES utf8
       sql_query               = SELECT id,name,companyname,radians(lat) as lat,radians(lng) as lng,salary,education,jobtype,secondtype,UNIX_TIMESTAMP(ctime) As ctime,UNIX_TIMESTAMP(mtime) As mtime FROM job WHERE id>(SELECT max_doc_id FROM sph_counter_product WHERE counter_id=1 )
       sql_query_post_index    = REPLACE INTO sph_counter_product SELECT 1,MAX(id) FROM job
    }
    #####index 定义
    index main
    {
            source  = main
            path    = /usr/local/sphinx/var/data/mysql
            docinfo = extern
            mlock   = 0
            morphology      = none
            min_word_len    = 1
            html_strip      = 0
            charset_dictpath     = /usr/local/mmseg3/etc/
            charset_type        = zh_cn.utf-8
     }
    index delta : main
    {
        source          = delta
        path            = /usr/local/sphinx/var/data/delta
    }
    #全局index定义
    indexer
    {
        mem_limit            = 256M
    }
    searchd
    {
        listen              = 9312
        read_timeout        = 5
        max_children        = 30
        max_matches         = 1000
        seamless_rotate     = 0
        preopen_indexes     = 0
        #attr_flush_period = 300
        #compat_sphinxql_magics        = 0 (默认不监听)
        unlink_old          = 1
        pid_file         = /usr/local/sphinx/var/log/searchd_mysql.pid   #请修改为实际使用的绝对路径，例如：/usr/local/coreseek/var/...
        log             = /usr/local/sphinx/var/log/searchd_mysql.log        #请修改为实际使用的绝对路径，例如：/usr/local/coreseek/var/...
        query_log         = /usr/local/sphinx/var/log/query_mysql.log    #请修改为实际使用的绝对路径，例如：/usr/local/coreseek/var/...
       }
      上面为数据源配置
      调用命令列表：
      执行索引（查询、测试前必须执行一次）


     /usr/local/sphinx/bin/indexer -c /usr/local/sphinx/etc/sphinx.conf --all --rotate
     启动后台服务（必须开启）


     /usr/local/sphinx/bin/searchd -c /usr/local/sphinx/etc/sphinx.conf
     启动报错 找不到mysql.sock 解决方法：
     使用软链接 ln -s  /tmp/mysql.sock /var/lib/mysql/mysql.sock
     执行增量索引


     /usr/local/sphinx/bin/indexer -c /usr/local/sphinx/etc/sphinx.conf delta --rotate
     合并索引


     /usr/local/sphinx/bin/indexer -c /usr/local/sphinx/etc/sphinx.conf --merge main delta --rotate --merge-dst-range deleted 0 0
     (为了防止多个关键字指向同一个文档加上--merge-dst-range deleted 0 0)
     后台服务测试


     /usr/local/coreseek/bin/search -c /usr/local/coreseek/etc/csft_mysql.conf  aaa
     关闭后台服务


     /usr/local/sphinx/bin/searchd -c /usr/local/sphinx/etc/sphinx.conf --stop
     自动化命令：


     crontab -e
     */1 * * * * /bin/sh /usr/local/sphinx/bin/indexer -c /usr/local/sphinx/etc/sphinx.conf delta --rotate
     */5 * * * * /bin/sh /usr/local/sphinx/bin/indexer -c /usr/local/sphinx/etc/sphinx.conf --merge main delta --rotate --merge-dst-range deleted 0 0
     30 1 * * *  /bin/sh /usr/local/sphinx/bin/indexer -c /usr/local/sphinx/etc/sphinx.conf --all --rotate
     以下任务计划的意思是：每隔一分钟执行一遍增量索引，每五分钟执行一遍合并索引，每天1:30执行整体索引。
     分布式配置


     index dist
     {
              type                            = distributed
              agent                           = 127.0.0.1:9313:index_3307_0
              agent                           = 127.0.0.1:9313:index_3307_0_delta
              agent                           = 127.0.0.1:9314:index_3307_1
              agent                           = 127.0.0.1:9314:index_3307_1_delta
              agent                           = 127.0.0.1:9316:index_3308_0
              agent                           = 127.0.0.1:9316:index_3308_0_delta
              agent                           = 127.0.0.1:9317:index_3308_1
              agent                           = 127.0.0.1:9317:index_3308_1_delta
              agent                           = 127.0.0.1:9319:index_3309_0
              agent                           = 127.0.0.1:9319:index_3309_0_delta
              agent                           = 127.0.0.1:9320:index_3309_1
              agent                           = 127.0.0.1:9320:index_3309_1_delta
              agent_query_timeout             = 100000
     }
    indexer
     {
         mem_limit           = 1024M
     }
    searchd
     {
         listen              = 9312
         read_timeout        = 5
         max_children        = 30
         max_matches         = 6000
         seamless_rotate     = 1
         preopen_indexes     = 1
         unlink_old          = 1
         compat_sphinxql_magics=0
         query_log_format    = sphinxql
         pid_file            = /usr/local/coreseek/var/log/searchd_mysql.pid
         log                 = /usr/local/coreseek/var/log/searchd_mysql.log
         query_log           = /usr/local/coreseek/var/log/query_mysql.log
        #workers            = threads
         dist_threads = 6  #几台分布式机器
     }
    最后记得加入到开机命令中：
    vi /etc/rc.local
    query(“销售”, “dist”); //dist主的分布式机器索引名
