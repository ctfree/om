
#1. Ganglia介绍

Ganglia是一个集群监控工具，由UC Berkeley创建并开源。Ganglia的中文意思是神经中枢，现在支持多部分操作系统（包括linux、unix、windows），可支持2000个节点的网络监控（当然这不是上限，只是一个大集群使用的范例）。

**进程组件**

Ganglia底层使用RRDTool获得数据，Ganglia主要分三个进程组件：

*   gmond（ganglia monitor deamon） 每个被检测的节点或集群运行一个gmond进程，进行监控数据的收集、汇总和发送。gmond即可以作为发送者（收集本机数据），也可以作为接收者（汇总多个节点的数据）

*   gmetad（ganglia metadata deamon） 通常在整个监控体系中只有一个gmetad进程。该进程定期检查所有的gmonds，主动收集数据，并存储在RRD存储引擎中。

*   ganglia-web 使用php编写的web界面，以图表的方式展现存储在RRD中的数据。通常与gmetad进程运行在一起。

**部署结构**

Ganglia的层次化结构做的非常好，由小到大可以分为node -&gt; cluster -&gt; grid，这三个层次。

*   一个**node**就是一个需要监控的节点，一般是个主机，用IP表示。每个node上运行一个gmond进程用来采集数据，并提交给gmetad。

*   一个**cluster**由多个node组成，就是一个集群，我们可以给集群定义名字。一个集群可以选一个node运行gmetad进程，汇总/拉取gmond提交的数据，并部署web front，将gmetad采集的数据用图表展示出来。

*   一个**grid**由多个cluster组成，是一个更高层面的概念，我们可以给grid定义名字。grid中可以定义一个顶级的gmetad进程，汇总/拉取多个gmond、子gmetad提交的数据，部署web front，将顶级gmetad采集的数据用图表展示出来。

显然，这种方式非常灵活，可以实现多种结构的数据监控。由下图，我们可以清晰的看出这种层次化的结构，和不同的部署方式。

![](http://images.cnitblog.com/blog/128843/201307/19174753-350fa64c955e4943a1f4c185d1cdeaea.jpg)

![](http://img3.ph.126.net/rrxXCDjTURb16yF-QV2_-Q==/650488671195299903.png)

#2. 安装

安装脚本如下：  


```bash

	version=3.6.1
	
	install_3rd()
	{
	  yum -y install apr-devel apr-util check-devel cairo-devel pango-devel libxml2-devel \
	rpmbuild glib2-devel dbus-devel freetype-devel fontconfig-devel gcc-c++ expat-devel \
	python-devel libXrender-devel pcre pcre-devel
	}
	
	install_confuse()
	{
	  cd /usr/local/src
	  wget http://savannah.nongnu.org/download/confuse/confuse-2.7.tar.gz
	  tar -xzvf confuse-2.7.tar.gz
	  cd confuse-2.7
	  ./configure CFLAGS=-fPIC --disable-nls --prefix=/usr/local/confuse
	  make
	  make install
	  mkdir -p /usr/local/confuse/lib64
	  cp -a -f /usr/local/confuse/lib/* /usr/local/confuse/lib64/
	  echo "/usr/local/confuse/lib" >> /etc/ld.so.conf.d/my-lib.conf
	
	}
	
	install_ganglia()
	{
	  cd /usr/local/src
	  wget  http://downloads.sourceforge.net/project/ganglia/ganglia%20monitoring%20core/$version/ganglia-$version.tar.gz
	  tar -xzvf ganglia-$version.tar.gz
	  cd ganglia-$version
	  ./configure --prefix=/usr/local/ganglia --with-librrd=/usr/local/rrdtool --with-libconfuse=/usr/local/confuse --enable-gexec --enable-status --with-gmetad --sysconfdir=/etc/ganglia
	  make
	  make install
	
	}
	
	install_rrd()
	{
	
	  cd /usr/local/src
	  wget http://oss.oetiker.ch/rrdtool/pub/rrdtool-1.4.9.tar.gz
	  tar -xzvf rrdtool-1.4.9.tar.gz
	  cd rrdtool-1.4.9
	  ./configure --prefix=/usr/local/rrdtool-1.4.9
	  make
	  make install
	  ln -s /usr/local/rrdtool-1.4.9 /usr/local/rrdtool
	  echo /usr/local/rrdtool/lib >> /etc/ld.so.conf.d/my-lib.conf 
	  ldconfig
	  ln -s /usr/local/rrdtool/lib/librrd.so.4.2.2 /usr/lib/librrd.so
	}
	
	install_web()
	{
	  cd /usr/local/src
	  wget http://jaist.dl.sourceforge.net/project/ganglia/ganglia-web/3.6.2/ganglia-web-3.6.2.tar.gz
	  tar -vxf ganglia-web-3.6.2.tar.gz
	  cd ganglia-web-3.6.2
	  sed -i -e "s@GDESTDIR = /usr/share/ganglia-webfrontend@GDESTDIR = /var/www/html/ganglia@"  Makefile
      sed -i -e "s@APACHE_USER = www-data@APACHE_USER = apache@"  Makefile
	  make install
	  chown -R apache:apache /var/www/html/ganglia
	}
	
	
	regist_gmetad()
	{
	  cd /usr/local/src/ganglia-$version
	  cp -f gmetad/gmetad.init /etc/init.d/gmetad
	  cp -f /usr/local/ganglia/sbin/gmetad /usr/sbin/gmetad
	  chkconfig --add gmetad
	  service gmetad start
	}
	
	regist_gmond()
	{
	  cd /usr/local/src/ganglia-$version
	  cp -f gmond/gmond.init /etc/init.d/gmond
	  cp -f /usr/local/ganglia/sbin/gmond /usr/sbin/gmond
	  chkconfig --add gmond
	  gmond --default_config > /etc/ganglia/gmond.conf
	  service gmond start
	
	}
	
	print_usage() {
	  echo "Usage         : ./install.sh [gmetad|gmond]"
	}
	
	# Parse parameters
	while [ $# -gt 0 ]; do
	  case "$1" in
	    gmetad )
	      shift
	      echo "install ganglia gmetad..."
	      install_3rd
	      install_confuse
	      install_rrd
	      install_ganglia
		  install_web
		  regist_gmetad
	      regist_gmond
	      echo "changed=yes comment='install ganglia gmetad ok'"
	      exit
	      ;;
	    gmond )
	      shift
	      echo "install ganglia gmond..."
	      install_3rd
	      install_confuse
	      install_rrd
	      install_ganglia
	      regist_gmond
	      echo "changed=yes comment='install ganglia gmond ok'"
	      exit
	      ;;              
	    *)  
	      print_usage
	      echo "changed=no comment='Unknown argument: $1'"
	      exit
	      ;;
	    esac
	  shift
	done
	print_usage
	echo "changed=no comment='Unknown argument: $1'"

```

*  在主节点上安装ganglia，并启动gmetad 服务：

```
    ./install.sh gmetad
```

*  在被监控节点上安装ganglia，并启动gmond 服务：

```
    ./install.sh gmond
```

安装完成后可以通过http://ip:port/ganglia 访问查看监控信息  
ip： 主节点ip  
port：apache 服务端口  



#3. 配置

*  主节点上配置gmetad：打开/etc/ganglia/gmetad.conf 添加数据源 和 网格名称：

```
	data_source "my cluster" localhostt 
	gridname "MyGrid"
```

*  被监控节点上配置gmond：打开/etc/ganglia/gmond.conf 修改 cluster name ：  

```
cluster { 
    name = "my cluster"
    owner = "fonsview"
    latlong = "unspecified"
    url = "unspecified" 
}
```

*  配置 ganglia-web,打开/etc/httpd/conf.d/ganglia.conf：
```
	Alias /ganglia /var/www/html/ganglia  // /var/www/html/ganglia 是ganglia 的安装路径

	<Directory /ganglia>
	        AllowOverride All
	        Order allow,deny
	        Allow from all   
	        Deny from none
	</Directory>

```


*  服务启动
```
   service gmetad start // 启动gmetad 服务
   service gmond start  // 启动gmond 服务
   service http reload  // 启动ganglia-web 服务
```

#4. 扩展

Ganglia 的扩展性很强，可以很方便使用其提供的一些机制来创建自定义的
metric，以收集应用数据等。它主要提供了以下方式：

1. Python 扩展
2. gmetric 命令

而 ganglia 的数据格式和协议是完全开放的，第三方应用也完全可以按照其格
式发送 metric 数据给 gmond。Ganglia 官方也将一些
[第三方扩展收集到一起了](https://github.com/ganglia/gmond_python_modules)
，可以用来监控诸如redis、MySQL 等等应用，可以根据需要选用。

这里通过实际中添加的wcache监控,来说明如何使用ganglia python 扩展

要增加一个扩展需要修改三个地方：  

1.  python 脚本 位于/usr/local/ganglia/lib64/ganglia/python_modules/
2.  pyconf 配置 位于/etc/ganglia/conf.d/
3.  graph  配置 位于/var/www/html/ganglia/graph.d/

下面逐一介绍

##4.1 python脚本

python 脚本用于向gmond添加自定义的模块，模块中必须包含的三个方法是：

def metric_init(params):

def metric_cleanup():

def metric_handler(name):

前面两个方法的名字必须是一定的，而最后一个 metric_handler可以任意命名。

**\__main\__**是便于debug用，可以单独调试该模块，以检测是否有错。

下面将对每个方法的功能做解释。

**def metric_init(params)**:
对模块的初始化，在gmond服务被启动的时候，运行一次。

该方法必须返回一个词典列表，每个词典表示了一个metric的信息。每个词典的格式如下：

```python
d1 = {'name': 'wcache_ClientHttpRequests_5m',             #metric的名字
        'call_back': get_stat,    #收集到数据后调用的方法
        'time_max': 60,              
        'value_type': 'float',         #string | uint | float | double 
        'units': 'N',                 # metric的单位 在ganglia-web 上可以看到
        'slope': 'both',              #zero | positive | negative | both
        'format': '%f',               #必须和value_type对应(http://docs.python.org/library/stdtypes.html#string-formatting)
        'description': 'Example module metric (random numbers)',    #对metric的描述，在前端可以看到
        'groups': 'wcache'}           #这个metric属于的组,如果没有定义，会分到no_group metric中

```

**def metric_cleanup()**:
gmond关掉的时候执行，不能返回值。

**def metric_handler(name)**:
可以取任何的名字，要在call_back中调用，使用的是get_stat。

/usr/local/ganglia/lib64/ganglia/python_modules/wcache.py 脚本较长， 
见附录 wcache.py。




##4.2 pyconf 
pyconf是python模块的配置文件:

**Modules**:对每个模块进行配置  
**name**:模块名,同时必须与创建的python文件名一致  
**language**: 语言  
**param**:参数列表，所有的参数作为一个dict(即map)传给python脚本的metric_init(params)函数。  


**collection_group**:需要收集的metric列表，一个模块中可以扩展任意个metric  
**collect_every**: 汇报周期，以秒为单位。  
**metric**：可以有多个，定义每个metric的信息。  

/etc/ganglia/conf.d/wcache.pyconf代码如下

```python

#/* wcache server metrics */
#
modules {
  module {
    name = "wcache"
    language = "python"
  }
}

collection_group {
  collect_every = 30
  time_threshold = 60
  metric {
    name = wcache_ClientHttpRequests_5m
    title = "Client Http Per Second requests 5m"
  }
  metric {
    name = wcache_ClientHttpErrors_5m
    title = "Client Http Per Second errors 5min"
  }
  metric {
    name = wcache_ClientHttpRequests_60m
    title = "Client Http Per Second requests 60m"
  }
  metric {
    name = wcache_ClientHttpErrors_60m
    title = "Client Http Per Second errors 60min"
  }
  metric {
    name = wcache_ReqHitRatio_5m
    title = "Hits as % of all requests 5min"
  }
  metric {
    name = wcache_MemHitRatio_5m
    title = "Memory Hits as % of hit requests 5min"
  }
  metric {
    name = wcache_DiskHitRatio_5m
    title = "Disk Hits as % of hit requests 5min"
  }
  metric {
    name = wcache_ByteHitRatio_5m
    title = "Hits as % of bytes sent 5min"
  }
  metric {
    name = wcache_ReqHitRatio_60m
    title = "Hits as % of all requests 60min"
  }
  metric {
    name = wcache_MemHitRatio_60m
    title = "Memory Hits as % of hit requests 60min"
  }
  metric {
    name = wcache_DiskHitRatio_60m
    title = "Disk Hits as % of hit requests 60min"
  }
  metric {
    name = wcache_ByteHitRatio_60m
    title = "Hits as % of bytes sent 60min"
  }
  metric {
    name = wcache_HttpRequestNumber
    title = "Number of HTTP requests received"
  }
  metric {
    name = wcache_RequestFailureRatio
    title = "Request failure ratio"
  }
  metric {
    name = wcache_StorageSwapCapacity
    title = "Storage Swap capacity"
  }
  metric {
    name = wcache_StorageMemCapacity
    title = "Storage Mem capacity"
  }
  metric {
    name = wcache_MeanObjectSize
    title = "Mean Object Size"
  }
  metric {
    name = wcache_ServiceTime
    title = "Median Service Times"
  }
  metric {
    name = wcache_HitServiceTime
    title = "Hit Median Service Times"
  }
  metric {
    name = wcache_MissServiceTime
    title = "Miss Median Service Times"
  }
  metric {
    name = wcache_StoreEntries
    title = "StoreEntries"
  }
  metric {
    name = wcache_MemStoreEntries
    title = "StoreEntries with MemObjects"
  }
}


```



##4.3 graph 配置

数据收集完成后，还需要将数据已图表的形式展示在Ganglia web界面中。所以需要配置前端数据展示方式

为了将几个相同的属性放置到一张report图表中，配置如下

/var/www/html/ganglia/graph.d/wcache_HitRatio_1h_report.json

```python
{
    "report_name": "wcache_HitRatio_1h_report",
    "report_type": "standard",
    "title": "wcache 1 hour HitRatio",
    "vertical_label": "precent",
    "series": [{
        "metric": "wcache_ReqHitRatio_1h",
        "color": "3366ff",
        "label": "ReqHitRatio",
        "line_width": "2",
        "type": "line"
    }, {
        "metric": "wcache_ByteHitRatio_1h",
        "color": "cc99ff",
        "label": "ByteHitRatio",
        "line_width": "2",
        "type": "line"
    }, {
        "metric": "wcache_MemHitRatio_1h",
        "color": "ff3366",
        "label": "MemHitRatio",
        "line_width": "2",
        "type": "line"
    }, {
        "metric": "wcache_DiskHitRatio_1h",
        "color": "00ff00",
        "label": "DiskHitRatio",
        "line_width": "2",
        "type": "line"
    }]
}

```


配置完成后，report图表还不会显示到ganglia-web 上，需要修改
/var/lib/ganglia-web/conf/default.json,将wcache_HitRatio_1h_report 加入
```json
{
    "included_reports": ["load_report", "mem_report", "cpu_report", "multicpu_idle_report", "network_report", "wcache_HitRatio_1h_report"]
}

```


所有脚本配置完成后，重新启动gmond 服务。观察前端web数据，查看模块添加是否成功 。


#5. 附录
##5.1 wcache.py
<span id="jump">wcache.py</span>
```python

import sys
import os
import re

import time
import logging

log=logging.getLogger("wcache");
log.setLevel(logging.DEBUG)

ch =logging.FileHandler("/tmp/gmond_wcache.log")
ch.setLevel(logging.DEBUG)
formatter = logging.Formatter("%(asctime)s - %(name)s - %(levelname)s\t Thread-%(thread)d - %(message)s")
ch.setFormatter(formatter)

log.addHandler(ch)
log.debug('starting up')

last_update = 0
elapsed_time = 0
# We get counter values back, so we have to calculate deltas for some stats
squid_stats = {}
squid_stats_last = {}

PREFIX="wcache_"
PREFIX_LEN=len(PREFIX)
MIN_UPDATE_INTERVAL = 10          # Minimum update interval in seconds

commonds=[]

class Commond:
    def __init__(self,commond):
        self.commond=commond
        self.stats_descriptions={}

    def addStat(self,metric,stats_description):
        self.stats_descriptions[metric]=stats_description



def parseCommond(commond):
    log.debug('parseCommond '+commond)
    global elapsed_time
    try:
        squidclient = os.popen(commond)
    except IOError,e:
        log.error('error running '+commond)
        return False

    # Parse output, splitting everything into key/value pairs
    rawstats = {}
    for stat in squidclient.readlines():
        stat = stat.strip()
        if stat.find(':') >= 0:
            [key,value] = stat.split(':',1)
            if value:     # Toss things with no value
                value = value.lstrip()
                rawstats[key.strip()] = value
        elif stat.find('=') >= 0:
            [key,value] = stat.split('=',1)
            if value:     # Toss things with no value
                value = value.lstrip()
                rawstats[key.strip()] = value
        else:
            match = re.search("(\d+)\s+(.*)$", stat) # reversed "value key" line
            if match:
                rawstats[match.group(2).lstrip()] = match.group(1).lstrip()
    log.debug(str(rawstats))
    # Use stats_descriptions to convert raw stats to real metrics
    for metric in stats_descriptions:
        if stats_descriptions[metric]['commond'] != commond:
            continue;
        if stats_descriptions[metric].has_key('key'):
            if rawstats.has_key(stats_descriptions[metric]['key']):
                log.debug("metric key: "+metric)
                rawstat = rawstats[stats_descriptions[metric]['key']]
                if stats_descriptions[metric].has_key('match'):
                    match = re.match(stats_descriptions[metric]['match'], rawstat)
                    if match:
                        rawstat = match.group(1)
                        log.debug("match: "+stats_descriptions[metric]['match']+" rawstat:"+rawstat);
                        squid_stats[metric] = rawstat
                else:
                    squid_stats[metric] = rawstat

                if squid_stats.has_key(metric): # Strip trailing non-num text
                    if metric != 'cacheVersionId': # version is special case
                        match = re.match('([0-9.]+)',squid_stats[metric]);
                        squid_stats[metric] = float(match.group(1))
                        if stats_descriptions[metric]['type'] == 'integer':
                            squid_stats[metric] = int(squid_stats[metric])

                # Calculate delta for counter stats
                # if metric in squid_stats_last:
                #     if stats_descriptions[metric]['type'] == 'counter32':
                #         current = squid_stats[metric]
                #         log.debug("elapsed_time: %f"%elapsed_time)
                #         squid_stats[metric] = (squid_stats[metric] - squid_stats_last[metric]) / float(elapsed_time)
                #         squid_stats_last[metric] = current
                #     else:
                #         squid_stats_last[metric] = squid_stats[metric]
                # else:
                #     if metric in squid_stats:
                #         squid_stats_last[metric] = squid_stats[metric]
            log.debug('collect_stats done')
            log.debug('squid_stats: ' + str(squid_stats))
    return True

def collect_stats():
    log.debug('collect_stats()')
    global last_update,elapsed_time
    global squid_stats, squid_stats_last

    now = time.time()

    if now - last_update < MIN_UPDATE_INTERVAL:
        log.debug(' wait ' + str(int(MIN_UPDATE_INTERVAL - (now - last_update))) + ' seconds')
        return True
    else:
        elapsed_time = now - last_update
        last_update = now

    squid_stats = {}
    for commond in commonds:
        parseCommond(commond);
    return True;


def get_stat(name):
    log.info("get_stat(%s)" % name)
    global squid_stats

    ret = collect_stats()

    if ret:
        if name.startswith(PREFIX):
            label = name[PREFIX_LEN:]
        else:
            lable = name

        log.debug("fetching %s" % label)
        try:
            log.info("got %4.2f" % squid_stats[label])
            return squid_stats[label]
        except:
            log.error("failed to fetch %s" % name)
            return 0

    else:
        return 0


def metric_init(params):
    global descriptors
    global squid_stats
    global stats_descriptions   # needed for stats extraction in collect_stat()

    log.debug("init: " + str(params))
    stats_descriptions = dict(

        ClientHttpRequests_5m={
            'description':'Client Http Per Second requests 5m',
            'units':'requests/s',
            'type': 'float',
            'key': 'client_http.requests',
            'match': '([0-9.]+)/sec',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p 80 mgr:5min'
        },

        ClientHttpErrors_5m={
            'description':'Client Http Per Second errors 5min',
            'units':'requests/s',
            'type': 'float',
            'key': 'client_http.errors',
            'match': '([0-9.]+)/sec',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p 80 mgr:5min'
        },

        ClientHttpRequests_60m={
            'description':'Client Http Per Second requests 60m',
            'units':'requests/s',
            'type': 'float',
            'key': 'client_http.requests',
            'match': '([0-9.]+)/sec',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p 80 mgr:60min'
        },

        ClientHttpErrors_60m={
            'description':'Client Http Per Second errors 60min',
            'units':'requests/s',
            'type': 'float',
            'key': 'client_http.errors',
            'match': '([0-9.]+)/sec',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p 80 mgr:60min'
        },

        ReqHitRatio_5m={
            'description':'Hits as % of all requests 5min',
            'units':'percent',
            'type': 'float',
            'key': 'Hits as % of all requests',
            'match': '5min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        MemHitRatio_5m={
            'description':'Memory Hits as % of hit requests 5min',
            'units':'percent',
            'type': 'float',
            'key': 'Memory hits as % of hit requests',
            'match': '5min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        DiskHitRatio_5m={
            'description':'Disk Hits as % of hit requests 5min',
            'units':'percent',
            'type': 'float',
            'key': 'Disk hits as % of hit requests',
            'match': '5min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        ByteHitRatio_5m={
            'description':'Hits as % of bytes sent 5min',
            'units':'percent',
            'type': 'float',
            'key': 'Hits as % of bytes sent',
            'match': '5min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        ReqHitRatio_60m={
            'description':'Hits as % of all requests 60min',
            'units':'percent',
            'type': 'float',
            'key': 'Hits as % of all requests',
            'match': '5min: [0-9.]+%,\s+60min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        MemHitRatio_60m={
            'description':'Memory Hits as % of hit requests 60min',
            'units':'percent',
            'type': 'float',
            'key': 'Memory hits as % of hit requests',
            'match': '5min: [0-9.]+%,\s+60min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        DiskHitRatio_60m={
            'description':'Disk Hits as % of hit requests 60min',
            'units':'percent',
            'type': 'float',
            'key': 'Disk hits as % of hit requests',
            'match': '5min: [0-9.]+%,\s+60min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        ByteHitRatio_60m={
            'description':'Hits as % of bytes sent 60min',
            'units':'percent',
            'type': 'float',
            'key': 'Hits as % of bytes sent',
            'match': '5min: [0-9.]+%,\s+60min: ([0-9.]+)%',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        HttpRequestNumber={
            'description':'Number of HTTP requests received',
            'units':'requests',
            'type': 'float',
            'key': 'Number of HTTP requests received',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        RequestFailureRatio={
            'description':'Request failure ratio',
            'units':'percent',
            'type': 'float',
            'key': 'Request failure ratio',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        StorageSwapCapacity={
            'description':'Storage Swap capacity',
            'units':'percent',
            'type': 'float',
            'key': 'Storage Swap capacity',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        StorageMemCapacity={
            'description':'Storage Mem capacity',
            'units':'percent',
            'type': 'float',
            'key': 'Storage Mem capacity',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        MeanObjectSize={
            'description':'Mean Object Size',
            'units':'KB',
            'type': 'float',
            'key': 'Mean Object Size',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        ServiceTime={
            'description':'Median Service Times',
            'units':'seconds',
            'type': 'float',
            'key': 'HTTP Requests (All)',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        HitServiceTime={
            'description':'Hit Median Service Times',
            'units':'seconds',
            'type': 'float',
            'key': 'Cache Hits',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        MissServiceTime={
            'description':'Miss Median Service Times',
            'units':'seconds',
            'type': 'float',
            'key': 'Cache Misses',
            'match': '([0-9.]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        StoreEntries={
            'description':'StoreEntries',
            'units':'objects',
            'type': 'integer',
            'key': 'StoreEntries',
            'match': '([0-9]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },

        MemStoreEntries={
            'description':'StoreEntries with MemObjects',
            'units':'objects',
            'type': 'integer',
            'key': 'StoreEntries with MemObjects',
            'match': '([0-9]+)',
            'commond':'/opt/fonsview/NE/webcache/bin/webcacheclient -p80 mgr:info'
        },
    )

    for label in stats_descriptions:
        if commonds.count(stats_descriptions[label]["commond"]) ==0:
            commonds.append(stats_descriptions[label]["commond"])

    descriptors = []
    # collect_stats()

    #time.sleep(MIN_UPDATE_INTERVAL)
    #collect_stats()


    for label in stats_descriptions:
        #if squid_stats.has_key(label):
        if stats_descriptions[label]['type'] == 'string':
            d = {
                'name': PREFIX + label,
                'call_back': get_stat,
                'time_max': 60,
                'value_type': "string",
                'units': '',
                'slope': "none",
                'format': '%s',
                'description': label,
                'groups': 'wcache',
            }
        elif stats_descriptions[label]['type'] == 'counter32':
            d = {
                'name': PREFIX + label,
                'call_back': get_stat,
                'time_max': 60,
                'value_type': "float",
                'units': stats_descriptions[label]['units'],
                'slope': "positive",
                'format': '%f',
                'description': label,
                'groups': 'wcache',
            }
        elif stats_descriptions[label]['type'] == 'integer':
            d = {
                'name': PREFIX + label,
                'call_back': get_stat,
                'time_max': 60,
                'value_type': "uint",
                'units': stats_descriptions[label]['units'],
                'slope': "both",
                'format': '%u',
                'description': label,
                'groups': 'wcache',
            }
        else:
            d = {
                'name': PREFIX + label,
                'call_back': get_stat,
                'time_max': 60,
                'value_type': "float",
                'units': stats_descriptions[label]['units'],
                'slope': "both",
                'format': '%f',
                'description': label,
                'groups': 'wcache',
            }

        d.update(stats_descriptions[label])

        descriptors.append(d)

        # else:
        #     log.error("skipped " + label)
    log.info("descriptors:"+str(len(descriptors))+" "+str(descriptors))
    return descriptors


def metric_cleanup():
    logging.shutdown()
    pass

#This code is for debugging and unit testing
if __name__ == '__main__':
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)

    ch = logging.StreamHandler(sys.stdout)
    ch.setLevel(logging.DEBUG)
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    ch.setFormatter(formatter)
    root.addHandler(ch)
    metric_init(None)
    for d in descriptors:
        v = d['call_back'](d['name'])
        if d['value_type'] == 'string':
            print 'value for %s is %s %s' % (d['name'], v, d['units'])
        elif d['value_type'] == 'uint':
            print 'value for %s is %d %s' % (d['name'], v, d['units'])
        else:
            print 'value for %s is %4.2f %s' % (d['name'], v, d['units'])

```
