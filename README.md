# sersync
```
文件同步(rsync+inotify+sersync)
注意：此操作步骤为单向同步，如需双向同步，反向再设置一次即可
```


# 主机侧
```
#!/bin/bash
#Active Server
syncUserName=dxhysync
remoteAddress=10.0.0.13
syncDir=/home/test
failLogFilePath=/home/log/rsync/rsync_fail_log.sh
remoteModelName=xwikiData


yum install xinetd rsync gcc -y

echo "123456" > /etc/rsyncPwd/remoteRsync.pwd

chmod 600 /etc/rsyncPwd/remoteRsync.pwd
mkdir -p /home/install
cd /home/install

wget --no-check-certificate https://sourceforge.net/projects/inotify-tools/files/inotify-tools/3.13/inotify-tools-3.13.tar.gz
tar -zxvf inotify-tools-3.13.tar.gz
cd inotify-tools-3.13
./configure
make && make install

wget --no-check-certificate https://raw.githubusercontent.com/orangle/sersync/master/release/sersync2.5.4_64bit_binary_stable_final.tar.gz
tar xf sersync2.5.4_64bit_binary_stable_final.tar.gz
mv GNU-Linux-x86/* /usr/local/sersync/
mv /usr/local/sersync/confxml.xml /usr/local/sersync/confxml.xml_bak

ln -sf /usr/local/sersync/sersync2 /usr/bin/

cat << EOF > /usr/local/sersync/confxml.xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<head version="2.5">
  <host hostip="localhost" port="8008"></host>
  <debug start="true"/>
  <fileSystem xfs="false"/>
  <filter start="false">
    <exclude expression="(.*)\.php"></exclude>
    <exclude expression="^data/*"></exclude>
  </filter>
  <inotify>
    <delete start="true"/>
    <createFolder start="true"/>
    <createFile start="false"/>
    <closeWrite start="true"/>
    <moveFrom start="true"/>
    <moveTo start="true"/>
    <attrib start="false"/>
    <modify start="false"/>
  </inotify>
 
  <sersync>
    <!-- 这里填写服务器A要同步的文件夹路径-->
    <localpath watch="$syncDir"> 
      <!-- 这里填写服务器B的IP地址和模块名-->
      <remote ip="$remoteAddress" name="$remoteModelName"/> 
    </localpath>
    <rsync>
    <commonParams params="-artuz"/>
    <!-- rsync+密码文件 这里填写服务器B的认证信息-->
    <auth start="true" users="$syncUserName" passwordfile="/etc/rsyncPwd/remoteRsync.pwd"/> 
    <!-- port=874 -->
    <userDefinedPort start="false" port="874"/>
    <!-- timeout=100 -->
    <timeout start="false" time="100"/>
    <ssh start="false"/>
    </rsync>
    <!--default every 60mins execute once-->
    <!-- 修改失败日志记录(必选)-->
    <failLog path="$failLogFilePath" timeToExecute="60"/>
    <!--600mins-->
    <!-- 定时任务,执行全量同步 -->
    <crontab start="true" schedule="240">
      <crontabfilter start="false">
        <exclude expression="*.php"></exclude>
        <exclude expression="info/*"></exclude>
      </crontabfilter>
    </crontab>
    <plugin start="false" name="command"/>
  </sersync>
 
  <!-- 下面这些有关于插件你可以忽略了 -->
  <plugin name="command">
    <param prefix="/bin/sh" suffix="" ignoreError="true"/> <!--prefix /opt/tongbu/mmm.sh suffix-->
    <filter start="false">
      <include expression="(.*)\.php"/>
      <include expression="(.*)\.sh"/>
    </filter>
  </plugin>
 
  <plugin name="socket">
  <localpath watch="/home/demo">
  <deshost ip="210.36.158.xxx" port="8009"/>
  </localpath>
  </plugin>

  <plugin name="refreshCDN">
    <localpath watch="/data0/htdocs/cdn.markdream.com/site/">
      <cdninfo domainname="cdn.chinacache.com" port="80" username="xxxx" passwd="xxxx"/>
      <sendurl base="http://cdn.markdream.com/cms"/>
      <regexurl regex="false" match="cdn.markdream.com/site([/a-zA-Z0-9]*).cdn.markdream.com/images"/>
    </localpath>
  </plugin>
</head>
EOF

echo '已完成'
sersync2 -d -o /usr/local/sersync/confxml.xml
echo 'sersync 启动完成!'
echo "sersync2 -d -o /usr/local/sersync/confxml.xml" >> /etc/rc.local
echo 'sersync 开机自启设置完成'
```

# 备机侧
```
#Backup Server
syncUserName=dxhysync
localAddress=10.0.0.13
allowAddressPrefix=10.0.0
syncDir=/home/test
logFilePath=/home/log/rsync/rsyncd.log

#install rsync
yum install xinetd rsync -y
#add a user use in sync
useradd $syncUserName && echo '123456' | passwd --stdin $syncUserName
#create a password file and assign 600 right
mkdir -p /etc/rsyncPwd
echo $syncUserName':123456'  >> /etc/rsyncPwd/localRsync.pwd
chmod 600 /etc/rsyncPwd/localRsync.pwd

#overwrite config of rsync
cat << EOF > /etc/rsyncd.conf
uid = root
gid = root
address = $localAddress
hosts allow = $allowAddressPrefix.0/24
use chroot = yes
max connections = 100
pid file = /var/run/rsync.pid
lock file  = /var/run/rsync.lock
log file = $logFilePath 
motd file = /etc/rsyncd.motd

[xwikiData]
path =$syncDir
comment = used for active recording files
read only = false
list = yes
auth users = $syncUserName
secrets file =/etc/rsyncPwd/localRsync.pwd
hosts allow = $allowAddressPrefix.1/255.255.255.0
EOF

#temporary settings some arguments
sysctl -w fs.inotify.max_queued_events="99999999"
sysctl -w fs.inotify.max_user_watches="99999999"
sysctl -w fs.inotify.max_user_instances="65535"

#permanent settings some arguments
echo "fs.inotify.max_queued_events = 99999999" >> /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 65535" >> /etc/sysctl.conf
echo "fs.inotify.max_user_watches = 99999999" >> /etc/sysctl.conf

sysctl -p

#start rsync daemon
rsync --daemon
#add rsync to system start
echo "/usr/bin/rsync --daemon --config=/etc/rsyncd.conf" >> /etc/rc.local
```
