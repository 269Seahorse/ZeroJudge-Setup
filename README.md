# ZeroJudge Ubuntu 22.04 架設

## 前言

目前 ZeroJudge 只有 [GitHub 上的 repo](https://github.com/jiangsir/ZeroJudge)，雖然的原始碼是完整的，但是 README.md 裡面只有打包好的 ova 檔案。因此，我試著在自己的 Server 架設了 ZeroJudge，成功之後打算寫個說明，讓大家參考。目前只有試過 Ubuntu 22.04 AMD64，其他 OS 和 Arm 架構都是還沒試過的，大家試過後可以回報是否適用。建議以新安裝的 OS 架設。
## 架設步驟

### 1. 以 root 登入你的 Server

```powershell=
ssh root@<Server IP>
```

如果 SSH port 不是 22

```powershell=
ssh root@<Server IP> -p <port>
```

### 2. 安裝必要套件

```bash=
sudo apt update
sudo apt upgrade -y
sudo apt install python3 ant python3-pip mysql-server mysql-client-core-8.0 libguestfs-tools git lxc nano tar default-jdk wget -y
```

### 3. 新增一個新的 user 並給予 sudo 權限

因為 ZeroJudge 需要還要讓他可以 no password sudo

```bash=
sudo useradd -r -s /bin/bash -m <user>
sudo passwd <user>
nano /etc/sudoers
```

最後一行加上

```text
<user> ALL=(ALL:ALL) NOPASSWD:ALL
```

### 4. 設定 MySQL

```bash=
mysql -uroot -p
# 更改 root 密碼
ALTER USER 'root'@'localhost' IDENTIFIED BY '<password>';
exit;
mysql -uroot -p
# 新增一個 mysql 用戶
CREATE USER '<user>'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
# ZeroJudge 用的資料庫
CREATE DATABASE <database name>;
GRANT ALL ON <database name>.* TO '<user>'@'localhost';
FLUSH PRIVILEGES;
exit;
```

### 5. 切換成剛剛建立的 user 的帳號

```powershell=
exit
ssh <user>@<Server IP>
```

### 6. 安裝 Tomcat

```bash=
sudo groupadd tomcat
sudo useradd -s /bin/false -g tomcat -d /var/lib/tomcat8 tomcat
cd /tmp
# 從我的 repo 下載 Tomcat，版本過舊官方網站 404
wget https://github.com/269Seahorse/ZeroJudge-Setup/raw/main/apache-tomcat-8.5.73.tar.gz
sudo mkdir /var/lib/tomcat8
sudo tar xzvf apache-tomcat-8.5.73.tar.gz -C /var/lib/tomcat8 --strip-components=1
cd /var/lib/tomcat8
sudo chgrp -R tomcat /var/lib/tomcat8
sudo chmod -R g+r conf
sudo chmod g+x conf
sudo chown -R tomcat webapps/ work/ temp/ logs/
sudo update-java-alternatives -l
```

你會看到下列這種輸出

```shell
java-1.11.0-openjdk-amd64      1111       /usr/lib/jvm/java-1.11.0-openjdk-amd64
```

接下來建立檔案

```bash=
sudo nano /etc/systemd/system/tomcat8.service
```

輸入 (Environment=JAVA_HOME 這行為上面輸出結果後面加 " /jre")

```service
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64 /jre
Environment=CATALINA_PID=/var/lib/tomcat8/temp/tomcat.pid
Environment=CATALINA_HOME=/var/lib/tomcat8
Environment=CATALINA_BASE=/var/lib/tomcat8
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/var/lib/tomcat8/bin/startup.sh
ExecStop=/var/lib/tomcat8/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

完成之後要 reload

```bash=
sudo systemctl daemon-reload
```

開始 Tomcat

```bash=
sudo systemctl start tomcat8
```

確認是否正確執行

```bash=
sudo systemctl status tomcat8
```

新增使用者(管理Web GUI頁面)

```bash=
sudo nano /var/lib/tomcat8/conf/tomcat-users.xml
```

在 \<tomcat-users> 跟 </tomcat-users> 之間加上

```xml
<role rolename="manager-gui" />
<user username="manager" password="<password>" roles="manager-gui" />
<role rolename="admin-gui" />
<user username="admin" password="<password>" roles="manager-gui,admin-gui" />
```

然後編輯 Webapp 管理頁面

```bash=
sudo nano /var/lib/tomcat8/webapps/manager/META-INF/context.xml
sudo nano /var/lib/tomcat8/webapps/host-manager/META-INF/context.xml
```

找到

```xml
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
```

在前後補上 <!\-- \-->

```xml
<!--<Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />-->
```

最後重開

```bash=
sudo systemctl restart tomcat8
```

打開 `http://<server ip>:8080` 應該長這樣就對了
![image](https://github.com/269Seahorse/ZeroJudge-Setup/blob/main/tomcat.png?raw=true)

### 7. 獲取 ZeroJudge 需要的 LXC 檔案

先回去 home

```bash=
cd
```

然後安裝 gdown

```bash=
pip install gdown
export PATH="/home/<user>/.local/bin:$PATH"
source ~/.profile
# 確認 /home/<user>/.local/bin 是否有效
echo $PATH
```

下載 ZeroJudge 虛擬機

```bash=
gdown https://drive.google.com/uc?id=1Y-PW031lYHOuxqQUe3rjL2B8ePEGyRuC
```

解壓縮虛擬機

```bash=
tar -xvf ZeroJudgeVM3.4.2.ova
```

掛接 vmdk 並複製 lxc 檔案

```bash=
sudo mkdir /mnt/u1
sudo guestmount -i -r /mnt/u1 -a ZeroJudgeVM-disk001.vmdk
sudo cp -r /mnt/u1/var/lib/lxc/lxc-ALL/ /var/lib/lxc/
sudo service lxc restart
sudo umount /mnt/u1
```

清理檔案

```bash=
rm ZeroJudgeVM*
```

### 8. 給 tomcat 權限

```bash=
nano /etc/sudoers
```

最後一行加上

```text
tomcat ALL=(ALL:ALL) NOPASSWD:ALL
```

### 9. 架設 ZeroJudge

先安裝套件

```bash=
sudo pip install fire bs4 lxml
```

複製 tomcat library 到 /usr/share/tomcat8

```bash=
sudo mkdir /usr/share/tomcat8
sudo cp -r /var/lib/tomcat8/lib/ /usr/share/tomcat8/
```

clone ZeroJudge 的 repo

```bash=
git clone https://github.com/jiangsir/ZeroJudge.git
```

開始執行

```bash=
sudo python3 ZeroJudge/setup.py install <dbpass> --dbuser <dbuser>
```

由於預設剛開始要本機登入，所以要先修改允許 IP

```bash=
sudo mysql -uroot -p
USE zerojudge;
UPDATE appconfigs set manager_ip = '[0.0.0.0/0]';
UPDATE appconfigs set allowedIP = '[0.0.0.0/0]';
```

for Judge Server

```bash=
sudo -i
nano /var/lib/tomcat8/webapps/ZeroJudge_Server/WEB-INF/ServerConfig.xml
```

輸入

```xml=
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
<comment></comment>
<entry key="cryptKey">ZZEERROO</entry>
<entry key="ServerInfo"></entry>
<entry key="Servername">ZeroJudgeServer</entry>
<entry key="Lxc_EXEC">sudo lxc-attach -n lxc-ALL</entry>
<entry key="ServerOS">UbuntuServer</entry>
<entry key="version">3.5</entry>
<entry key="isTLEbreak">true</entry>
<entry key="sshport">22</entry>
<entry key="allowIPset">[0.0.0.0/0]</entry>
<entry key="JVM_MB">3000</entry>
<entry key="rsyncAccount">zero</entry>
<entry key="Lxc_PATH">/var/lib/lxc/lxc-ALL/rootfs/</entry>
<entry key="isCleanTmpFile">true</entry>
<entry key="CONSOLE_PATH">/JudgeServer_CONSOLE</entry>
<entry key="Compilers">[{"version":"gcc -std=c11 (gcc 7.4.0)","path":"","language":"C","command_begin":"","cmd_compile":"gcc $S/$C.c -std=c11 -lm -lcrypt -O2 -pipe -DONLINE_JUDGE -o $S/$C.exe","cmd_namemangling":"nm -A $S/$C.exe","cmd_execute":"$S/$C.exe &lt; $T &gt; $S/$C.out","cmd_makeobject":"g++ $S/$C.c -o $S/$C.o","timeextension":1.0,"command_end":"","samplecode":"#include&lt;stdio.h&gt;\r\nint main() {\r\n char s[9999];\r\nwhile( scanf(\"%s\",s)!=EOF ) {\r\n printf(\"hello, %s\\n\",s);\r\n }\r\n return 0;\r\n}","restrictedfunctions":["system","fopen","fclose","freopen","fflush","fstream","time.h","#pargma","conio.h","fork","popen","execl","execlp","execle","execv","execvp","getenv","putenv","setenv","unsetenv","socket","connect","fwrite","gethostbyname"],"enable":"C","suffix":"c"},{"version":"g++ -std=c++14 (g++ 7.4.0)","path":"","language":"CPP","command_begin":"","cmd_compile":"g++ -std=c++14 -lm -lcrypt -O2 -pipe -DONLINE_JUDGE -o $S/$C.exe $S/$C.cpp","cmd_namemangling":"nm -A $S/$C.exe","cmd_execute":"$S/$C.exe &lt; $T &gt; $S/$C.out","cmd_makeobject":"g++ $S/$C.cpp -o $S/$C.o","timeextension":1.0,"command_end":"","samplecode":"#include &lt;iostream&gt;\r\nusing namespace std;\r\n\r\nint main() {\r\nstring s;\r\n while(cin &gt;&gt; s){\r\ncout &lt;&lt; \"hello, \"&lt;&lt; s &lt;&lt; endl;\r\n }\r\n return 0;\r\n}","restrictedfunctions":["system","fopen","fclose","freopen","fflush","fstream","ifstream","ofstream","time.h","#pargma","conio.h","fork","popen","execl","execlp","execle","execv","execvp","getenv","putenv","setenv","unsetenv","socket","connect","fwrite","gethostbyname"],"enable":"CPP","suffix":"cpp"},{"version":"OpenJDK java version 1.8.0_222","path":"","language":"JAVA","command_begin":"","cmd_compile":"javac -encoding UTF-8 $S/$C.java","cmd_namemangling":"javap -classpath $S -verbose $C","cmd_execute":"java -Dfile.encoding=utf-8 -classpath $S $C &lt; $T &gt; $S/$C.out","cmd_makeobject":"","timeextension":3.0,"command_end":"","samplecode":"import java.util.Scanner;\r\npublic class JAVA {\r\n\tpublic static void main(String[] args) {\r\n\t\tScanner cin= new Scanner(System.in);\r\n\t\tString s;\r\n\t\twhile (cin.hasNext()) {\r\n\t\t\ts=cin.nextLine();\r\n\t\t\tSystem.out.println(\"hello, \" + s);\r\n\t\t}\r\n\t}\r\n}","restrictedfunctions":["java\\.io\\.File.*","java\\.net\\..*","java\\.lang\\.Thread","java\\.lang\\.Runtime","java\\.lang\\.Runnable","java\\.lang\\.Process","java\\.applet\\..*","java\\.awt\\..*","java\\.nio\\..*","java\\.sql\\..*","java\\.security\\..*","java\\.rmi\\..*","java\\.lang\\.Exception","java\\.lang\\.RuntimeException"],"enable":"JAVA","suffix":"java"},{"version":"Free Pascal Compiler version 3.0.4","path":"","language":"PASCAL","command_begin":"","cmd_compile":"fpc -o$S/$C.exe -Sg $S/$C.pas","cmd_namemangling":"nm -A $S/$C.o","cmd_execute":"$S/$C.exe &lt; $T &gt; $S/$C.out","cmd_makeobject":"","timeextension":1.0,"command_end":"","samplecode":"var s : string;\r\nbegin\r\nwhile not eof do begin\r\nreadln(s);\r\nwriteln('hello, ',s);\r\nend;\r\nend.","restrictedfunctions":["SYSTEM_RESET","SYSTEM_CLOSE","SYSTEM_ASSIGN","SYSTEM_REWRITE","dos","SysUtils","exec","stdcall","external","assign","reset","rewrite","close","execute","OBJPAS_CLOSEFILE","OBJPAS_ASSIGNFILE","GenerateInstruction_CALL_FAR","SysProc_SeekFile","SysProc_EraseFile","SysProc_RenameFileC","SysProc_TruncFile"],"enable":null,"suffix":"pas"},{"version":"Python 3.6.9","path":"","language":"PYTHON","command_begin":"","cmd_compile":"","cmd_namemangling":"","cmd_execute":"PYTHONIOENCODING=utf-8 python3 $S/$C.py &lt; $T &gt; $S/$C.out","cmd_makeobject":"","timeextension":3.0,"command_end":"","samplecode":"while True:\r\n    try:\r\n        s = input()\r\n        print('hello, '+s)\r\n    except:\r\n        break\r\n","restrictedfunctions":[],"enable":"PYTHON","suffix":"py"}]</entry>
</properties>
```

然後

```bash=
service tomcat8 restart
exit
```

打開 http://\<server ip>:8080，用管理員帳號登入之後修改裁判機IP (http://127.0.0.1:8080/ZeroJudge_Server/) 就可以了
管理員預設帳密

```text=
帳號 zero
密碼 !@#$zerojudge
```

最後權限要給 zero/tomcat

```bash=
sudo chown -R zero:tomcat /ZeroJudge_CONSOLE/
sudo chown -R zero:tomcat /JudgeServer_CONSOLE/
```

接下來你用 Python 測試 ZeroJudge 的時候你會發現這個錯誤: /shell.exe: /lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.34' not found (required by /shell.exe)

修復方法：

```bash=
sudo -i
git clone https://github.com/matrix1001/glibc-all-in-one.git
cd glibc-all-in-one/
nano update_list #把最上面的 python 改成 python3
./update_list
cat list
# 你會看到可以下載的列表
./download <挑一個能用的> # 範例 ./download 2.35-0ubuntu3_amd64
cp libs/<你剛剛選的版本>/* /var/lib/lxc/lxc-ALL/rootfs/lib/x86_64-linux-gnu/
service lxc restart
exit
```

恭喜，你完成 ZeroJudge 的架設了！
