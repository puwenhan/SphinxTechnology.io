# SphinxTechnology.io
Sphinx 全文索引引擎
1.  什么是Sphinx ? Sphinx 中文意为全文索引引擎,但是只支持中文和俄文,但是国人开发了中文包软件,coreseek.
2. 为什么需要Sphinx? 因为Mysql数据库在执行匹配操作的时候无法使用到任何的索引优化,导致要查询的数据量比较大,查询时间比较长.如果要根据歌词查歌曲，
   或者根据电影情节查电影的名字,会消耗太多的时间。一般人们都会使用第三方软件,如Sphinx，lucence.
3.Mysql本身支持全文索引的功能,但是存在两个问题,a.只有Myisam 引擎支持;b.对中文的支持不太友好.
4.Sphinx的使用原理: (1) 创建数据源(也就是在数据库中建一张表,插入你准备要查询的数据,或者已经建好的表) (2) 对数据源建立索引( index 数据源),使用Sphinx的分词技术.
      (3)php 把要查询的关键字传递给Sphinx,然后Sphinx 根据关键字查找保存在Mysql中的id,最后Sphinx把id传给数据库，查找相应的数据.
5.Sphinx 目录下的文件夹列表: api,var,bin,etc(以及文件 readme.txt,test.txt,test_cjk.cmd,test_mysql.cmd,test_python.cmd).
6.使用:
      第一步:把etc目录下面的csft_mysql.conf文件拷贝到上一级目录，并改名为sphinx.conf.
      第二步:配置配置文件:
      # 源定义 contents 可以随意定义
source contents
{
    type                       = mysql

    sql_host                = localhost
    sql_user                = root
    sql_pass               = root
    sql_db                   = sphinx
    sql_port                = 3306
    sql_query_pre      = SET NAMES utf8

    # 第一必须是id 切是整数 后面的字段为需要添加索引的字段 
    sql_query                = SELECT  id,title, info FROM contents
}

# 索引文件定义 index NAME  (NAME 可以所以定义)
index contents
{
    source            = contents             #对应的source名称
    path            =  E:\server\sphinx\var\data\contents  # 索引文件存放路径
    
    # 以下无须更改
    docinfo            	 = extern
    mlock           	 = 0
    morphology      	 = none
    min_word_len        = 1
    html_strip               = 0
    # 以上无须更改

    # 中文字典位置
    charset_dictpath = E:\server\sphinx\etc                           
    charset_type        = zh_cn.utf-8
}

# 全局index定义 可以更改内存限制
indexer
{
    mem_limit            = 512M
}

#  服务端配置
searchd
{
    listen                  	=   9312
    read_timeout        = 5
    max_children        = 30
    max_matches       = 1000  #sphinx 返回ID数目
    seamless_rotate   = 0
    preopen_indexes  = 0
    unlink_old              = 1

    # 更改为对应的PID 日志路径
    pid_file = E:\server\sphinx\var\log\searchd_mysql.pid  
    log = E:\server\sphinx\var\log\searchd_mysql.log        
    query_log = E:\server\sphinx\var\log\query_mysql.log
}
第三步:创建索引，找到指定文件下(bin)的indexer.exe,到cmd窗口执行 indexer.exe -c (绝对路径下的)配置文件 --all|索引文件名.
   查看索引文件(Sphinx\var\data).
第四步：启动Sphinx的服务器: searched.exe 在cmd窗口下执行 -c (绝对路径下的)配置文件 --install|--stop|--delete.
第五步:通过php查询: 把api 文件下的sphinxapi.php的文件拷贝到项目文件下.(代码如下)
<?php
 header('Content-type:text/html;charset=utf8');
 require 'sphinxapi.php';
 //使用sphinx 完成查询
 $sc = new SphinxClient();
 //生成客户端，设置服务器
 $sc->setServer('localhost','9312');

 //var_dump($sc);die;
 //查询的关键字以及索引的文件
 $keyword = 'a';
 $indexName = 'contents';
 //查询我们需要内容
 $res = $sc->query($keyword,$indexName);
 //echo $sc->_error;
 $ids = $res['matches'];
 $id = array_keys($ids);
 //var_dump($id);die;
 $id = implode(',',$id);
 //var_dump($id);die;
 //连接数据库,查找数据
 mysql_connect('localhost:3306','root','root') or die("can't connect mysql");
 //选择数据库
 mysql_query('use sphinx') or die('failed to choose database!!');
 //设置字符集
 mysql_query('set names utf8');
 //查找数据
 $sql = "select id ,title ,info from contents where id in ($id)";
 //echo $sql;die;
 //执行sql语句
 $res = mysql_query($sql);//输出结果是一个资源集,需要我们遍历输出
 //var_dump($res);die;
 //定义一个空的数组
 $list = array();
 //对资源集遍历输出
 while($row = mysql_fetch_assoc($res)){
 	 $list[] = $row;
 }
 //再将数据遍历出来
 foreach ($list as $k => $v) {
 	echo $v['title'].'<br />'.$v['info'].'<hr />';
 }
  ?>
