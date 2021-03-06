# btcpool矿池-测试环境搭建及使用cgminer测试

本文档基于Ubuntu 16.04 LTS, 64 Bits。

## 安装Bitcoind+ZMQ

```shell
#Dependencies
apt-get -y install build-essential libtool autotools-dev automake autoconf pkg-config bsdmainutils python3
apt-get -y install libssl-dev libboost-all-dev libevent-dev
apt-get -y install libdb-dev libdb++-dev
apt-get -y install libminiupnpc-dev libzmq3-dev
apt-get -y install libqt5gui5 libqt5core5a libqt5dbus5 qttools5-dev qttools5-dev-tools libprotobuf-dev protobuf-compiler libqrencode-dev

#To Build
wget https://github.com/bitcoin/bitcoin/archive/v0.15.1.tar.gz
tar -zxvf bitcoin-0.15.1.tar.gz
cd bitcoin-0.15.1/
./autogen.sh
./configure --with-incompatible-bdb --prefix=/work/bitcoin
make
make install

#start/stop service
cd /work/bitcoin/bin/
./bitcoind --daemon -testnet -zmqpubhashtx=tcp://0.0.0.0:18331 -zmqpubhashblock=tcp://0.0.0.0:18331
#./bitcoin-cli -testnet stop
```

## 安装ZooKeeper

```shell
#Install ZooKeeper
apt-get install -y zookeeper zookeeper-bin zookeeperd

#mkdir for data
mkdir -p /work/zookeeper
mkdir /work/zookeeper/version-2
touch /work/zookeeper/myid
chown -R zookeeper:zookeeper /work/zookeeper

#set machine id
echo 1 > /work/zookeeper/myid

#edit config file
vim /etc/zookeeper/conf/zoo.cfg

initLimit=5
syncLimit=2
clientPort=2181
clientPortAddress=127.0.0.1
dataDir=/work/zookeeper
#伪分布式
server.1=127.0.0.1:2888:3888

#start/stop service
service zookeeper restart
#service zookeeper start/stop/restart/status
```

## 安装Kafka

```shell
#install depends
apt-get install -y default-jre

#install Kafka
mkdir /root/source
cd /root/source
wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/0.11.0.2/kafka_2.11-0.11.0.2.tgz
mkdir -p /work/kafka
cd /work/kafka
tar -zxf /root/source/kafka_2.11-0.11.0.2.tgz --strip 1

#edit conf
vim /work/kafka/config/server.properties

broker.id=1
offsets.topic.replication.factor=1
message.max.bytes=20000000
replica.fetch.max.bytes=30000000
log.dirs=/work/kafka-logs
listeners=PLAINTEXT://127.0.0.1:9092
#伪分布式
zookeeper.connect=127.0.0.1:2181

#start server
cd /work/kafka
nohup /work/kafka/bin/kafka-server-start.sh /work/kafka/config/server.properties > /dev/null 2>&1 &
```

## 安装BTCPool

```shell
#Build
cd /work
wget https://raw.githubusercontent.com/btccom/btcpool/master/install/install_btcpool.sh
bash ./install_btcpool.sh
```

如下内容为install_btcpool.sh展开：

```
CPUS=`lscpu | grep '^CPU(s):' | awk '{print $2}'`

apt-get update
apt-get install -y build-essential autotools-dev libtool autoconf automake pkg-config cmake
apt-get install -y openssl libssl-dev libcurl4-openssl-dev libconfig++-dev libboost-all-dev libmysqlclient-dev libgmp-dev libzookeeper-mt-dev

# zmq-v4.1.5
mkdir -p /root/source && cd /root/source
wget https://github.com/zeromq/zeromq4-1/releases/download/v4.1.5/zeromq-4.1.5.tar.gz
tar zxvf zeromq-4.1.5.tar.gz
cd zeromq-4.1.5
./autogen.sh && ./configure && make -j $CPUS
make check && make install && ldconfig

# glog-v0.3.4
mkdir -p /root/source && cd /root/source
wget https://github.com/google/glog/archive/v0.3.4.tar.gz
tar zxvf v0.3.4.tar.gz
cd glog-0.3.4
./configure && make -j $CPUS && make install

# librdkafka-v0.9.1
apt-get install -y zlib1g zlib1g-dev
mkdir -p /root/source && cd /root/source
wget https://github.com/edenhill/librdkafka/archive/0.9.1.tar.gz
tar zxvf 0.9.1.tar.gz
cd librdkafka-0.9.1
./configure && make -j $CPUS && make install

# libevent-2.0.22-stable
mkdir -p /root/source && cd /root/source
wget https://github.com/libevent/libevent/releases/download/release-2.0.22-stable/libevent-2.0.22-stable.tar.gz
tar zxvf libevent-2.0.22-stable.tar.gz
cd libevent-2.0.22-stable
./configure
make -j $CPUS
make install

# btcpool
mkdir -p /work && cd /work
git clone https://github.com/btccom/btcpool.git
cd /work/btcpool
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j $CPUS
```

## 启动BTCPool及cgminer测试btcpool

### 启动gbtmaker

```shell
#配置gbtmaker
cd /work/btcpool/build/
mkdir run_gbtmaker
cd run_gbtmaker/
ln -s ../gbtmaker
cp /work/btcpool/src/gbtmaker/gbtmaker.cfg ./
vim gbtmaker.cfg

gbtmaker = {
  rpcinterval = 5;
  is_check_zmq = true;
};
bitcoind = {
  zmq_addr = "tcp://127.0.0.1:18331";
  rpc_addr    = "http://127.0.0.1:18332";
  rpc_userpwd = "bitcoinrpc:xxxx";
};
kafka = {
  brokers = "127.0.0.1:9092";
};

#启动gbtmaker
cd /work/btcpool/build/run_gbtmaker/
mkdir log_gbtmaker
./gbtmaker -c ./gbtmaker.cfg -l ./log_gbtmaker &
tail -f log_gbtmaker/gbtmaker.INFO
```

### 启动jobmaker

```shell
#配置jobmaker
cd /work/btcpool/build/
mkdir run_jobmaker
cd run_jobmaker/
ln -s ../jobmaker
cp /work/btcpool/src/jobmaker/jobmaker.cfg ./
vim jobmaker.cfg

testnet = true;
jobmaker = {
  stratum_job_interval = 20;
  gbt_life_time = 90;
  empty_gbt_life_time = 15;
  file_last_job_time = "/work/btcpool/build/run_jobmaker/jobmaker_lastjobtime.txt";
  block_version = 0;
};
kafka = {
  brokers = "127.0.0.1:9092";
};
zookeeper = {
  brokers = "127.0.0.1:2181";
};
pool = {
  payout_address = "mi9vpXBWJ31WGpRU7n7VJQG4PvTndHBoCN";
  coinbase_info = "region1/Project BTCPool/";
};

#启动jobmaker
cd /work/btcpool/build/run_jobmaker/
mkdir log_jobmaker
./jobmaker -c ./jobmaker.cfg -l ./log_jobmaker &
tail -f log_jobmaker/jobmaker.INFO
```

### 启动sserver

```shell
#配置sserver
cd /work/btcpool/build/
mkdir run_sserver
cd run_sserver/
ln -s ../sserver
cp /work/btcpool/src/sserver/sserver.cfg ./
vim ./sserver.cfg

testnet = true;
kafka = {
  brokers = "127.0.0.1:9092";
};
sserver = {
  ip = "0.0.0.0";
  port = 3333;
  id = 1;
  file_last_notify_time = "/work/btcpool/build/run_sserver/sserver_lastnotifytime.txt";
  enable_simulator = false;
  enable_submit_invalid_block = false;
  share_avg_seconds = 10;
};
users = {
  list_id_api_url = "http://index.qubtc.com/apidemo.php";
};

#启动sserver
cd /work/btcpool/build/run_sserver/
mkdir log_sserver
./sserver -c ./sserver.cfg -l ./log_sserver &
tail -f log_sserver/sserver.INFO
```

### cgminer测试btcpool

```shell
#安装cgminer
cd /work/
apt-get -y install build-essential autoconf automake libtool pkg-config libcurl3-dev libudev-dev
apt-get -y install libusb-1.0-0-dev
git clone https://github.com/ckolivas/cgminer.git
cd cgminer
sh autogen.sh
./configure --enable-cpumining --disable-opencl
make

#cgminer测试
./cgminer --cpu-threads 3 -o stratum+tcp://127.0.0.1:3333 -u jack -p x

./cgminer --cpu-threads 3 --url 127.0.0.1:3333 --userpass jack:x
#./cgminer -o stratum+tcp://127.0.0.1:3333 -u jack -p x --debug --protocol-dump
#--debug，调试模式
#--protocol-dump，协议输出
```

### 启动blkmaker

```shell
#安装MySQL
待补充

#配置blkmaker
cd /work/btcpool/build/
mkdir run_blkmaker
cd run_blkmaker/
ln -s ../blkmaker
cp /work/btcpool/src/blkmaker/blkmaker.cfg ./
vim blkmaker.cfg

bitcoinds = (
{
  rpc_addr    = "http://127.0.0.1:18332";
  rpc_userpwd = "bitcoinrpc:xxxx";
}
);
kafka = {
  brokers = "127.0.0.1:9092";
};
pooldb = {
  host = "localhost";
  port = 3306;
  username = "develop";
  password = "iZ2ze3r0li2kgfvjkvs2xeZ";
  dbname = "bpool_local_db";
};

#启动blkmaker
cd /work/btcpool/build/run_blkmaker/
mkdir log_blkmaker
./blkmaker -c ./blkmaker.cfg -l ./log_blkmaker &
tail -f log_blkmaker/blkmaker.INFO
```

## 官方文档

* [Install ZooKeeper](https://github.com/btccom/btcpool/blob/master/docs/INSTALL-ZooKeeper.md)
* [Install Kafka](https://github.com/btccom/btcpool/blob/master/docs/INSTALL-Kafka.md)
* [Bitcoind Unix Build Notes](https://github.com/bitcoin/bitcoin/blob/master/doc/build-unix.md)
* [Install BTC.COM Pool](https://github.com/btccom/btcpool/blob/master/INSTALL.md)

## 扩展阅读

* [比特币挖矿——介绍](https://www.jianshu.com/p/06d9bd788357)
* [比特币挖矿——区块链技术](https://www.jianshu.com/p/a3f4b2b2d4fa)
* [比特币挖矿——钱包](https://www.jianshu.com/p/c3de6bd3d1e8)
* [比特币挖矿——控制器与矿机](https://www.jianshu.com/p/28139d6f32c3)
* [比特币挖矿——p2pool矿池](https://www.jianshu.com/p/ea1cc9cea3a3)
* [比特币挖矿——建立Kafka&ZooKeeper集群](https://www.jianshu.com/p/083b6192a505)
* [比特币挖矿——集群矿池btcpool](https://www.jianshu.com/p/af5dc2cab0a9)
* [Ubuntu: 用cgminer挖掘CPU比特币(bitcoin)](https://lesca.me/archives/mine-bitcoin-on-cpu-with-cgminer.html)