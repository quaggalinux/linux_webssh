# linux_webssh  
使用web方式访问linux服务器的ssh，支持websocket方式反代不影响域名原来的页面服务   
  
这个方案优势是支持Cloudflare的CDN方式，这样就妙用无穷了，利用CDN可以让IPv4的客户端用浏览器ssh到IPv6 only或IPv4 nat有公网IPv6的小鸡，另外利用CDN可以SSH连接到原来不能直接连接的小鸡   
   
缘于前段时间有机友问是否可以用web方式访问linux服务器的ssh，于是折腾一下，填平了一些坑，记录过程   
   
以下使用ubuntu为例子进行配置    
   
首先安装wssh,期间可能提示先要安装python3-pip或python-pip，如果没有提示就直接安装好了   
   
#pip install webssh   
   
如果有安装前提要求的按提示安装下面的包   
#apt install python3-pip  
或  
#apt install python-pip  
  
安装完成理论上可以使用了，这里假设各位机友自己能够安装nginx，certbot，python3-certbot-nginx等包，建议使用apt方式安装，然后自己可以写虚拟主机配置并签证书  
   
如果https的页面已经可以正常访问的话，编辑虚拟主机配置，加入下面的配置内容，就只是location /websshpath/的包含部分   
下面两行的反斜号"/"非常重要，漏写会导致页面不能显示并且报告404，而且控制台的log也会不断报告404  
location /websshpath/   
proxy_pass       http://127.0.0.1:8888/   
   
------nginx虚拟主机最终配置，签名后的！！！------   
   
server {  
    listen       80;  
    server_name  www.yourdomain.com;  
location /websshpath/ {  
    proxy_pass       http://127.0.0.1:8888/;  
    proxy_http_version         1.1;  
    proxy_read_timeout 3600;  
    proxy_set_header Upgrade   $http_upgrade;  
    proxy_set_header Connection "upgrade";  
    proxy_set_header Host      $http_host;  
    proxy_set_header X-Real-IP $remote_addr;  
    proxy_set_header X-Real-PORT $remote_port;  
    proxy_set_header X-Forwarded-For $remote_addr;  
    }  
  
    listen [::]:443 ssl; # managed by Certbot  
    ssl_certificate /etc/letsencrypt/live/www.yourdomain.com/fullchain.pem; # managed by Certbot  
    ssl_certificate_key /etc/letsencrypt/live/www.yourdomain.com/privkey.pem; # managed by Certbot  
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot  
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot  
  
}  
  
有CDN情况下，即CF开启橙色小云朵情况下，千万要注释掉下面3行，否则点connect按钮会显示websocket认证错误  
    proxy_set_header X-Real-IP $remote_addr;  
    proxy_set_header X-Real-PORT $remote_port;  
    proxy_set_header X-Forwarded-For $remote_addr;  
没有CDN情况下，即只是解析DNS，CF小云朵灰色情况下，千万要写上面3行  
  
检查nginx配置是否有误  
#nginx -t  
  
设置nginx开机自启  
#systemctl enable nginx  
  
如果修改了ngnix的配置需要用下面命令重启  
#systemctl restart nginx  
  
启动webssh测试，不带参数的话，缺省监听地址127.0.0.1，端口8888  
#wssh  
  
webssh就会在前台运行，不断显示log条目，这里仅是测试运行效果，下面会写成服务方式启动  
  
这时可以在任意浏览器上输入https://www.yourdomain.com/websshpath  
由于是https方式访问，所以安全性是保证的，当然大家不放心可以点浏览器地址栏的小锁头，确认证书是否是你之前签发的，又或是在CF的CDN情况下的cloudflare证书    
  
理论上应该可以打开一个web页面让你输入机器名字（一般输入127.0.0.1，也可以输入同一私有网段的其他服务器ip地址），ssh端口，用户名，密码，点connect的图标即可显示ssh终端登录后的画面，这个web页面也是支持私钥方式登录的，你可以选私钥文件提交  
  
确认能够使用后，在之前linux服务器的终端控制台打入CTRL-C，终止wssh的运行，然后按照下面步骤写成服务方式  
  
------配置webssh的Systemd服务启动方式------  
  
确保是root用户并在根目录,检查是否已经有system目录  
#ls -l /usr/lib/systemd  
  
没有的的话创建system目录  
#mkdir /usr/lib/systemd/system  
  
编辑webssh服务文件  
#nano /usr/lib/systemd/system/webssh.service  
  
写入以下内容  
  
[Unit]  
Description=webssh  
After=network.target  
  
[Service]  
TimeoutStartSec=30  
ExecStart=/usr/local/bin/wssh --wpintvl=1200  
ExecStop=/bin/kill $MAINPID  
  
[Install]  
WantedBy=multi-user.target  
  
保存并退出  
  
  
设置开机启动webssh   
#systemctl enable webssh  
  
启动webssh服务   
#systemctl start webssh   
   
检查服务状态   
#systemctl status webssh   
   
检查网络连接状态及wssh是否有在监听设定的8888端口   
   
#netstat -tunap   
    
完成后可以在任意浏览器上输入https://www.yourdomain.com/websshpath  

这样就不会影响域名原来的页面服务了。




