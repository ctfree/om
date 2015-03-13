

#原理
Ganglia是一个集群监控工具，由UC Berkeley创建并开源。Ganglia的中文意思是神经中枢，现在支持多部分操作系统（包括linux、unix、windows），可支持2000个节点的网络监控（当然这不是上限，只是一个大集群使用的范例）。

**基本结构**

Ganglia底层使用RRDTool获得数据，Ganglia主要分为两个进程组件：

*   gmond（**g**anglia **mon**itor **d**eamon）
*   gmetad（**g**anglia **meta**data **d**eamon）

其中，gmond运行在集群每个节点上，收集RRDTool产生的数据；gmetad运行在监控服务器上，收集每个gmond的数据。Ganglia还提供了一个PHP实现的web front end，一般使用Apache2作为其运行环境，通过Web Front可以看到直观的各种集群数据图表。

Ganglia的层次化结构做的非常好，由小到大可以分为node -&gt; cluster -&gt; grid，这三个层次。

*   一个**node**就是一个需要监控的节点，一般是个主机，用IP表示。每个node上运行一个gmond进程用来采集数据，并提交给gmetad。
*   一个**cluster**由多个node组成，就是一个集群，我们可以给集群定义名字。一个集群可以选一个node运行gmetad进程，汇总/拉取gmond提交的数据，并部署web front，将gmetad采集的数据用图表展示出来。
*   一个**grid**由多个cluster组成，是一个更高层面的概念，我们可以给grid定义名字。grid中可以定义一个顶级的gmetad进程，汇总/拉取多个gmond、子gmetad提交的数据，部署web front，将顶级gmetad采集的数据用图表展示出来。

显然，这种方式非常灵活，可以实现多种结构的数据监控。由下图，我们可以清晰的看出这种层次化的结构，和不同的部署方式。

![](http://images.cnitblog.com/blog/128843/201307/19174753-350fa64c955e4943a1f4c185d1cdeaea.jpg)

![](http://g.hiphotos.baidu.com/baike/c0%3Dbaike80%2C5%2C5%2C80%2C26/sign=565a7df0720e0cf3b4fa46a96b2f997a/d058ccbf6c81800adfd059ecb13533fa828b471f.jpg)

#安装
  安装脚本如下：

```bash
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

```

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
	  echo "Usage         : ./install.sh [master|minion]"
	}
	
	# Parse parameters
	while [ $# -gt 0 ]; do
	  case "$1" in
	    master )
	      shift
	      echo "install ganglia master..."
	      install_3rd
	      install_confuse
	      install_rrd
	      install_ganglia
	      regist_gmetad
	      regist_gmond
	      echo "changed=yes comment='install ganglia master ok'"
	      exit
	      ;;
	    minion )
	      shift
	      echo "install ganglia minion..."
	      install_3rd
	      install_confuse
	      install_rrd
	      install_ganglia
	      regist_gmond
	      echo "changed=yes comment='install ganglia minion ok'"
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



##配置


配置gmond：打开/etc/ganglia/gmond.conf 修改 cluster name ：

	cluster { 
    name = "swift"
    owner = "unspecified"
    latlong = "unspecified"
    url = "unspecified" 
	}
配置gmetad：打开/etc/ganglia/gmetad.conf 添加数据源 和 网格名称：

	data_source "swift" localhost 
	gridname "MyGrid"


