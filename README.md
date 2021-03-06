
需要下载的文件
fdfs 依赖：        libfastcommon-master.zip
fdfs 软件源码：    fastdfs-5.11.tar.gz
fdfs nginx模块：   fastdfs-nginx-module-master.zip
nginx相关文件：    zlib-1.2.11.tar.gz
nginx相关文件：    pcre-8.00.zip
nginx相关文件：    nginx-1.17.5.tar.gz

开始安装
```
yum -y install gcc
unzip libfastcommon-master.zip
cd libfastcommon-master/
./make.sh && ./make.sh install

tar -xf fastdfs-5.11.tar.gz
cd fastdfs-5.11
./make.sh && ./make.sh install


在/etc/init.d/ 目录下生成了两个服务相关脚本，便于停止 启动  fdfs_storaged fdfs_trackerd 服务

ls
fdfs_storaged fdfs_trackerd

cp -a /usr/local/src/fastdfs-5.11/conf/*  /etc/fdfs/

修改配置文件
vi /etc/fdfs/tracker.conf

# the tracker server port
port=22122
 
# the base path to store data and log files
base_path=/opt/fastdfs/tracker
 
# HTTP port on this tracker server
http.server_port=9270


sed -i "s^/home/yuqing/fastdfs^/opt/fastdfs/tracker^g" /etc/fdfs/tracker.conf
sed -i "s^http.server_port=8080^http.server_port=9270^g" /etc/fdfs/tracker.conf
sed -i "s^/home/yuqing/fastdfs^/opt/fastdfs/storage^g" /etc/fdfs/storage.conf
mkdir -p /opt/fastdfs/tracker
mkdir -p /opt/fastdfs/storage
mkdir -p /opt/fastdfs/client

vim /etc/fdfs/storage.conf

# storage所属的组 同组中的数据是相互备份的
group_name=group1
 
# the storage server port
port=23000
 
# the base path to store data and log files
base_path=/opt/fastdfs/storage
 
# store_path#, based 0, if store_path0 not exists, it's value is base_path
# the paths must be exist
store_path0=/opt/fastdfs/storage
#store_path1=/home/caibh/fdfs2
 
# tracker服务器，虽然是同一台机器上，但是不能写127.0.0.1。这项配置可以出现一次或多次,trackf服务器可有多台
tracker_server=172.16.6.56:22122
tracker_server=172.16.6.57:22122
 
# the port of the web server on this storage server
http.server_port=8888




sed -i "s^/home/yuqing/fastdfs^/opt/fastdfs/client^g" /etc/fdfs/client.conf
sed -i "s^http.tracker_server_port=80^http.tracker_server_port=9270^g" /etc/fdfs/client.conf
vim /etc/fdfs/client.conf：（客户端测试上传使用的，可以不配置）

# the base path to store log files
base_path=/opt/fastdfs/client
# tracker_server can ocur more than once, and tracker_server format is
# "host:port", host can be hostname or ip address
tracker_server=172.16.6.56:22122
#HTTP settings
http.tracker_server_port=9270



unzip -q  fastdfs-nginx-module-master.zip
cd fastdfs-nginx-module-master
cp -f src/mod_fastdfs.conf   /etc/fdfs/


vim  /etc/fdfs/mod_fastdfs.conf

url_have_group_name = true
# the base path to store log files
base_path=/tmp
 
# FastDFS tracker_server can ocur more than once, and tracker_server format is
# "host:port", host can be hostname or ip address
# valid only when load_fdfs_parameters_from_tracker is true
tracker_server=172.16.6.56:22122
 
# the port of the local storage server
# the default value is 23000
storage_server_port=23000
 
# the group name of the local storage server
group_name=group1
 
# store_path#, based 0, if store_path0 not exists, it's value is base_path
# the paths must be exist
# must same as storage.conf
store_path0=/opt/fastdfs/storage
#store_path1=/home/yuqing/fastdfs1

sed -i "s^/home/yuqing/fastdfs^/opt/fastdfs/storage^g" /etc/fdfs/mod_fastdfs.conf
开启nginx访问
sed -i "s^url_have_group_name = false^url_have_group_name = true^g" /etc/fdfs/mod_fastdfs.conf


启动服务
/etc/init.d/fdfs_trackerd start
/etc/init.d/fdfs_storaged start



服务启动后安装nginx

安装zlib
tar zxf zlib-1.2.11.tar.gz
cd  zlib-1.2.11
./configure &> /dev/null && make  &> /dev/null && make install  &> /dev/null 

安装pcre
unzip -q pcre-8.00.zip
cd pcre-8.00
./configure &> /dev/null && make  &> /dev/null && make install  &> /dev/null 


下载nginx编译安装，支持fastdfs模块
cd nginx-1.17.5
./configure --add-module=/usr/local/src/fastdfs/fastdfs-nginx-module-master/src

编译有可能会报错

-o objs/addon/src/ngx_http_fastdfs_module.o
/software/fastdfs-nginx-module-1.20/src/ngx_http_fastdfs_module.c
In file included from /software/fastdfs-nginx-module-1.20/src/common.c:26:0,
from /software/fastdfs-nginx-module-1.20/src/ngx_http_fastdfs_module.c:6:
/usr/include/fastdfs/fdfs_define.h:15:27: 致命错误：common_define.h：没有那个文件或目录
#include "common_define.h"


需要改fastdfs-nginx-module-master/src/config文件，两个位置，修改如下：
原文件    ngx_module_incs="/usr/local/include"
修改后    ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"
原文件    CORE_INCS="$CORE_INCS /usr/local/include"
修改后    CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"

使用sed命令，config文件需要自行查找
sed -i 's^ngx_module_incs="/usr/local/include"^ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"^g'  fastdfs-nginx-module-master/src/config
sed -i 's^CORE_INCS="$CORE_INCS /usr/local/include"^CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"^g'  fastdfs-nginx-module-master/src/config

修改完成后重新编译安装
./configure --add-module=/usr/local/src/fastdfs-nginx-module-master/src --with-pcre=/usr/local/src/pcre-8.00
make  &> /dev/null && make install  &> /dev/null 



安装完成后检测
/usr/local/nginx/sbin/nginx  -t
/usr/local/nginx/sbin/nginx: error while loading shared libraries: libpcre.so.0: cannot open shared object file: No such file or directory

检测报错，查找缺少哪个库
# ldd  `which /usr/local/nginx/sbin/nginx`
	linux-vdso.so.1 =>  (0x00007fffa0785000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f16f2cec000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f16f2ad0000)
	libcrypt.so.1 => /lib64/libcrypt.so.1 (0x00007f16f2899000)
	libfastcommon.so => /lib/libfastcommon.so (0x00007f16f2658000)
	libfdfsclient.so => /lib/libfdfsclient.so (0x00007f16f2441000)
	libpcre.so.0 => not found
	libz.so.1 => /lib64/libz.so.1 (0x00007f16f222b000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f16f1e5d000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f16f2ef0000)
	libfreebl3.so => /lib64/libfreebl3.so (0x00007f16f1c5a000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f16f1958000)


找到这个库在哪
find / -name libpcre.so.1
创建软连接
ln -s /usr/local/lib/libpcre.so.0 /lib64


将如下配置加入nginx.conf server模块，监听端口端口根据实际需求修改
vim /usr/local/nginx/conf/nginx.conf

        location ~ /group([0-9])/M00 {
            #设置模块
            ngx_fastdfs_module;
        }

拷贝可执行文件并启动
cp /usr/local/nginx/sbin/nginx  /usr/sbin/
nginx

测试上传,上传一张图片 IMG_0096.JPG

fdfs_upload_file /etc/fdfs/client.conf  IMG_0096.JPG
group1/M00/00/00/rBAIv13LphKAYE_4AAY-cNoG5Gw236.JPG  ##返回值
```
访问
https://172.16.8.193:800/group1/M00/00/00/rBAIv13LphKAYE_4AAY-cNoG5Gw236.JPG
