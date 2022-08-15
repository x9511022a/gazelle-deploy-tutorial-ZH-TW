# Gazelle部署手冊
## 前置需求
- Debian base System(Debian,Ubuntu)
- Postgresql 
- Java ，這邊建議使用 Zulu
## 安裝Java

這邊建議安裝 Zulu，為 openjdk 的一種
```shell
# 安裝必要的套件
sudo apt-get -q update
sudo apt-get -yq install gnupg curl 

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
sudo apt-get install zulu<X>-ca-jre-headless
```

## 安裝Jboss/Wildfly
### Jboss7
#### 需求
- java 7
- postgresql 9.x
#### 安裝步驟
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
### TestManagement
請到[Nexus](https://gazelle.ihe.net/nexus/index.html#nexus-search;gav~~gazelle-tm-ear*~~~)下載最新版，example:

- 下載本體
```shell
wget https://gazelle.ihe.net/nexus/service/local/repositories/releases/content/net/ihe/gazelle/tm/gazelle-tm-ear/6.6.0/gazelle-tm-ear-6.6.0.ear
```

- 下載sql檔，並解壓縮
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

- 更改 jboss 設定，將以下設定增加在 datasources 欄位底下
```xml
<datasource jta="true" enabled="true" use-java-context="true"
            pool-name="TestManagementDS" jndi-name="java:jboss/datasources/TestManagementDS">
    <connection-url>jdbc:postgresql://localhost:5432/gazelle</connection-url>
    <driver>postgresql</driver>
    <security>
        <user-name>gazelle</user-name>
        <password>gazelle</password>
    </security>
    <pool>
        <min-pool-size>1</min-pool-size>
        <max-pool-size>70</max-pool-size>
        <prefill>false</prefill>
        <use-strict-min>false</use-strict-min>
        <flush-strategy>FailingConnectionOnly</flush-strategy>
    </pool>
    <validation>
        <check-valid-connection-sql>select 1</check-valid-connection-sql>
        <validate-on-match>false</validate-on-match>
        <background-validation>false</background-validation>
        <use-fast-fail>false</use-fast-fail>
    </validation>
    <timeout>
        <idle-timeout-minutes>10</idle-timeout-minutes>
        <blocking-timeout-millis>30000</blocking-timeout-millis>
    </timeout>
    <statement>
        <prepared-statement-cache-size>30</prepared-statement-cache-size>
        <track-statements>false</track-statements>
    </statement>
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

- 將本體複製進 jboss 中
```shell
sudo cp gazelle-tm-ear-6.6.0.ear /usr/local/jboss7/standalone/deployments/gazelle-tm.ear
```

- 啟動 jboss
```shell
sudo systemctl start jboss7.service
```