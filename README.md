# Gazelle 部署手冊

## 前置需求

- Debian base System(Debian,Ubuntu)，這邊推薦使用 Debian 9
- Postgresql
- Java ，這邊建議使用 Zulu

## 安裝 Java

這邊建議安裝 Zulu，為 openjdk 的一種

```shell
# 安裝必要的套件
sudo apt-get -q update
sudo apt-get -yq install gnupg curl apt-transport-https unzip vim

# 加入 Azul 的公鑰
sudo apt-key adv \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys 0xB1998361219BD9C9

# 下載 Azul 的 repo
curl -O https://cdn.azul.com/zulu/bin/zulu-repo_1.0.0-3_all.deb

# 安裝 repo
sudo apt-get install ./zulu-repo_1.0.0-3_all.deb

# 更新 apt
sudo apt-get update

# 安裝 java ， <X> 替換為版本
sudo apt-get install zulu7-ca-jre-headless
sudo apt-get install zulu8-ca-jre-headless
```

## 安裝 Jboss/Wildfly

### Jboss7

#### 需求

- java 7
- postgresql 9.x

#### 安裝步驟

- 安裝 zulu7

```shell
sudo apt install zulu7-ca-jre-headless
```

- 下載 Jboss

```shell
wget -nv -O /tmp/jboss-as-7.2.0.Final.zip https://gazelle.ihe.net/jboss7/jboss-as-7.2.0.Final.zip
```

- 下載初始化 shell 腳本

```shell
wget -nv -O /tmp/init.d_jboss7 https://gazelle.ihe.net/jboss7/init.d_jboss7
```

- 創建 jboss user & group

```shell
sudo useradd jboss
sudo groupadd jboss-admin
sudo adduser jboss jboss-admin
```

- 將 jboss 安裝至 /usr/local 資料夾

```shell
cd /usr/local
sudo mv /tmp/jboss-as-7.2.0.Final.zip .
sudo unzip ./jboss-as-7.2.0.Final.zip
sudo ln -s jboss-as-7.2.0.Final jboss7
sudo chown -R jboss:jboss-admin /usr/local/jboss7/
sudo chmod -R 755 /usr/local/jboss-as-7.2.0.Final/
sudo mkdir /var/log/jboss7
sudo chown -R jboss:jboss-admin /var/log/jboss7/
sudo chmod -R g+w /var/log/jboss7/
```

- 安裝 shell 腳本，讓 jboss 開機啟動

```shell
sudo mv /tmp/init.d_jboss7 /etc/init.d/jboss7
sudo chmod +x /etc/init.d/jboss7
sudo update-rc.d jboss7 defaults
```

- 更新 shell 腳本中 JAVA_HOME 的位置

```shell
# 請打開 /etc/init.d/jboss7，並將 JAVA_HOME 設定改成下面的位置
/usr/lib/jvm/zulu7-ca-amd64/bin

# 並更新
sudo systemctl daemon-reload
```

## Deploy 各個系統

### Gazelle SSO

#### 前置需求

- 安裝 zulu8

```shell
sudo apt install zulu8-ca-jre-headless
```

- 安裝 tomcat8

```shell
sudo apt install tomcat8
```

- 設定 tomcat，新增 JAVA_HOME=/usr/lib/jvm/zulu8-ca-amd64/jre

```shell
sudo vi /etc/init.d/tomcat8
```

- 修改 server.xml 中的設定
```shell
sudo vim /var/lib/tomcat8/conf/server.xml
```

並將以下選項的註釋刪除，改成以下內容，port部分可以自行修改
```xml
<Connector port="8209"
           protocol="AJP/1.3"
           redirectPort="8443"
           secretRequired="false" />
```

- 刪除 tomcat 預設內容

```shell
sudo rm -rf /var/lib/tomcat8/webapps/ROOT
```

- 下載並部署 Gazelle SSO

```shell
cd /var/lib/tomcat8/webapps
sudo wget https://gazelle.ihe.net/apereo-cas-gazelle/cas.war
sudo mv cas.war sso.war
```

- 下載、設定 cas 設定檔

```shell
sudo su
cd /etc
wget https://gazelle.ihe.net/apereo-cas-gazelle/cas.tgz
tar zxvf cas.tgz
mkdir /etc/cas/log
chown -R tomcat8:tomcat8 cas
```

將以下設定檔中的 stretch.localdomain 改成自己的 FQDN

```shell
vi /etc/cas/config/cas.properties
```

並將以下 database 名稱改成與 TestManagement 相同的名稱，通常為 gazelle

```properties
cas.authn.attributeRepository.jdbc[0].url=jdbc:postgresql://localhost:5432/cas   gazelle
cas.authn.jdbc.query[0].url=jdbc:postgresql://localhost:5432/cas    gazelle
```

修改/etc/cas/config/log4j2.xml，在第一個<RollingFile\>的</Policies\>下中加入下面設定

```xml
            <DefaultRolloverStrategy max="5">
                <Delete basePath="${sys:cas.log.dir}">
                  <IfFileName glob="*.log" />
                  <IfLastModified age="7d" />
                </Delete>
            </DefaultRolloverStrategy>
```

- 建立 SSO 設定檔
  大部分 Gazelle Tools 會去/opt/gazelle/cas 裡讀取 SSO 設定檔

```shell
 sudo mkdir -p /opt/gazelle/cas
 sudo touch /opt/gazelle/cas/file.properties
 sudo chown -R jboss:jboss-admin /opt/gazelle/cas
 sudo chmod -R g+w /opt/gazelle/cas
```

- 為了連線到 tomcat 與 jboss，你需要在 Apache2 的設定檔中增加以下類似設定

```conf
<Location /sso>
   ProxyPass        ajp://localhost:8209/sso
   ProxyPassReverse ajp://localhost:8209/sso
</Location>
```

- 修復 log4j 漏洞
  在/usr/share/tomcat8/bin 中的 setenv.sh 中增加下列參數，若沒有此檔案就建立它

```shell
CATALINA_OPTS="$CATALINA_OPTS -Dlog4j2.formatMsgNoLookups=true"
```

### TestManagement

請到[Nexus](https://gazelle.ihe.net/nexus/index.html#nexus-search;gav~~gazelle-tm-ear*~~~)下載最新版，example:

- 下載 ear 檔

```shell
wget https://gazelle.ihe.net/nexus/service/local/repositories/releases/content/net/ihe/gazelle/tm/gazelle-tm-ear/6.6.0/gazelle-tm-ear-6.6.0.ear
```

- 下載 sql 檔，並解壓縮

```shell
wget https://gazelle.ihe.net/nexus/service/local/repositories/releases/content/net/ihe/gazelle/tm/gazelle-tm-ear/6.6.0/gazelle-tm-ear-6.6.0-sql.zip
unzip -d sql gazelle-tm-ear-6.6.0-sql.zip
```

- 創建資料庫並初始化，這是直接安裝 postgresql 的情況

```shell
sudo su postgresql
psql
postgres=# CREATE USER gazelle;
postgres=# CREATE DATABASE "your\_database" OWNER gazelle ENCODING UTF-8;
postgres=# ALTER USER gazelle WITH ENCRYPTED PASSWORD 'password';
postgres=# \q
exit

psql -U gazelle your_database < schema-X.X.X.sql
psql -U gazelle your_database < init-X.X.X.sql
```

- 下載工具包，並安裝

```shell
wget https://gazelle.ihe.net/nexus/service/local/repositories/releases/content/net/ihe/gazelle/tm/gazelle-tm-ear/6.6.0/gazelle-tm-ear-6.6.0-dist.zip
sudo unzip -d / gazelle-tm-ear-6.6.0-dist.zip
```

- 安裝 Graphviz

```shell
sudo apt install graphviz
```

- 更改 jboss 設定
```shell
sudo vim /usr/local/jboss7/standalone/configuration/standalone.xml
```
將以下設定增加在 <datasources\> 欄位底下

```xml
<datasource jta="true" enabled="true" use-java-context="true"
            pool-name="TestManagementDS" jndi-name="java:jboss/datasources/TestManagementDS">
    <connection-url>jdbc:postgresql://localhost:5432/gazelle</connection-url>
    <driver>postgresql</driver>
    <security>
        <user-name>gazelle</user-name>
        <password>gazelle</password>
    </security>
</datasource>
```

- 創建 SSO 設定

```shell
sudo mkdir -p /opt/gazelle/cas
sudo touch /opt/gazelle/cas/gazelle-tm.properties
sudo chown -R jboss:jboss-admin /opt/gazelle/cas
sudo chmod -R g+w /opt/gazelle/cas
```

將以下內容貼進 gazelle-tm.properties 內

```properties
casServerUrlPrefix = https://yourUrl.com/sso
casServerLoginUrl = https://yourUrl.com/sso/login
casLogoutUrl = https://yourUrl.com/sso/logout
service = https://yourUrl.com/gazelle
```

- 停止 jboss

```shell
sudo systemctl stop jboss7.service
```

- 將 ear 檔複製進 jboss 中

```shell
sudo cp gazelle-tm-ear-6.6.0.ear /usr/local/jboss7/standalone/deployments/gazelle-tm.ear
```

- 啟動 jboss

```shell
sudo systemctl start jboss7.service
```

### Proxy

- 下載 ear 檔

```shell
wget https://gazelle.ihe.net/nexus/service/local/repositories/releases/content/net/ihe/gazelle/proxy/gazelle-proxy-ear/5.0.4/gazelle-proxy-ear-5.0.4.ear
```
