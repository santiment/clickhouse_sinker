# clickhouse_sinker

[![Build Status](https://travis-ci.com/housepower/clickhouse_sinker.svg?branch=master)](https://travis-ci.com/housepower/clickhouse_sinker)
[![Go Report Card](https://goreportcard.com/badge/github.com/housepower/clickhouse_sinker)](https://goreportcard.com/report/github.com/housepower/clickhouse_sinker)

clickhouse_sinker is a sinker program that transfer kafka message into [ClickHouse](https://clickhouse.yandex/).

Refers to [design](./design.md) for how it works.

## Features

- Uses Native ClickHouse client-server TCP protocol, with higher performance than HTTP.
- Easy to use and deploy, you don't need write any hard code, just care about the configuration file
- Support multiple parsers: csv, fastjson, gjson.
- Support multiple Kafka client: sarama, kafka-go.
- Support multiple sinker tasks, each runs on parallel.
- Support multiply kafka and ClickHouse clusters.
- Bulk insert (by config `bufferSize` and `flushInterval`).
- Parse messages concurrently.
- Write batches concurrently.
- Every batch is sharded to a determined clickhouse node. Exit if loop write failed.
- Custom sharding policy (by config `shardingKey` and `shardingPolicy`).
- At least once delivery guarantee.
- Dynamic config management with Nacos.

## Supported data types

- [x] UInt8, UInt16, UInt32, UInt64, Int8, Int16, Int32, Int64
- [x] Float32, Float64
- [x] String
- [x] FixedString
- [x] Date, DateTime, DateTime64 (custom layout parser)
- [x] Array(UInt8, UInt16, UInt32, UInt64, Int8, Int16, Int32, Int64)
- [x] Array(Float32, Float64)
- [x] Array(String)
- [x] Array(FixedString)
- [x] Nullable
- [x] [ElasticDateTime](https://www.elastic.co/guide/en/elasticsearch/reference/current/date.html) => Int64 (2019-12-16T12:10:30Z => 1576498230)

## Install && Run

### By binary files (suggested)

Download the binary files from [release](https://github.com/housepower/clickhouse_sinker/releases), choose the executable binary file according to your env, modify the `conf` files, then run `./clickhouse_sinker --local-cfg-dir conf`

### By container image

Modify the `conf` files, then run `docker run --volume conf:/etc/clickhouse_sinker quay.io/housepower/clickhouse_sinker`

### By source

- Install Golang

- Go Get

```
go get -u github.com/housepower/clickhouse_sinker/...
```

- Build && Run

```
make build
## modify the config files, set the configuration directory, then run it
./dist/clickhouse_sinker --local-cfg-dir conf
```

## Configuration

Refers to how [integration test](./go.test.sh) use the [example config](./conf/config.json).
Also refers to [code](./config/config.go) for all config items.

### Authentication with Kafka

clickhouse_sinker support following three authentication mechanism:

* [x] No authentication

An example kafka config:

```
    "kfk1": {
      "brokers": "127.0.0.1:9092",
      "Version": "0.10.2.1"
    }
```

* [x] SASL/PLAIN

An example kafka config:
```
    "kfk2": {
      "brokers": "127.0.0.1:9093",
      "sasl": {
        "enable": true,
        "password": "username",
        "username": "password"
      "Version": "0.10.2.1"
    }
```

* [x] SASL/GSSAPI(Kerberos)

You need to configuring a Kerberos Client on the host on which the clickhouse_sinker run. Refers to [Kafka Security](https://kafka.apache.org/documentation/#security).

1. Install the krb5-libs package on all of the client machine.
```
$ sudo yum install -y krb5-libs
```
2. Supply a valid /etc/krb5.conf file for each client. Usually this can be the same krb5.conf file used by the Kerberos Distribution Center (KDC).


An example kafka config:

```
    "kfk3": {
      "brokers": "127.0.0.1:9094",
      "sasl": {
        "enable": true,
        "gssapi": {
          "authtype": 2,
          "keytabpath": "/home/keytab/zhangtao.keytab",
          "kerberosconfigpath": "/etc/krb5.conf",
          "servicename": "kafka",
          "username": "zhangtao/localhost",
          "realm": "ALANWANG.COM"
        }
      },
      "Version": "0.10.2.1"
    }
```

FYI. The same config looks like the following in Java code:
```
security.protocol：SASL_PLAINTEXT
sasl.kerberos.service.name：kafka
sasl.mechanism：GSSAPI
sasl.jaas.config：com.sun.security.auth.module.Krb5LoginModule required useKeyTab=true storeKey=true debug=true keyTab=\"/home/keytab/zhangtao.keytab\" principal=\"zhangtao/localhost@ALANWANG.COM\";
```

## Configuration Management

### Nacos

Sinker is able to register with Nacos, get and apply config changes.
Controled by:

- CLI parameters: `nacos-register-enable, nacos-addr, nacos-namespace-id, nacos-group, nacos-username, nacos-password`
- env variables: `NACOS_REGISTER_ENABLE, NACOS_ADDR, NACOS_NAMESPACE_ID, NACOS_GROUP, NACOS_USERNAME, NACOS_PASSWORD`

### Consul

Currently sinker is able to register with Consul, but not able to get config.
Controled by:

- CLI parameters: `consul-register-enable, consul-addr, consul-deregister-critical-services-after`
- env variables: `CONSUL_REGISTER_ENABLE, CONSUL_ADDR, CONSUL_DEREGISTER_CRITICAL_SERVICES_AFTER`

### Local Files
TODO. Currently sinker is able to parse local config files at startup, but not able to detect file changes.

## Prometheus Metrics

All metrics are defined in `statistics.go`. You can create Grafana dashboard for clickhouse_sinker by importing the template `clickhouse_sinker-dashboard.json`.

* [x] Pull with prometheus

Metrics are exposed at `http://ip:port/metrics`. IP is the outbound IP of this machine. Port is from CLI `--http-port` or env `HTTP_PORT`.

Sinker registers with Nacos if CLI `--consul-register-enable` or env `CONSUL_REGISTER_ENABLE` is present. However Prometheus is [unable](https://github.com/alibaba/nacos/issues/1032) to obtain dynamic service list from nacos server.

* [x] Push to promethues

If CLI `--push-gateway-addrs` or env `PUSH_GATEWAY_ADDRS` (a list of comma-separated urls) is present, metrics are pushed to the given one of given URLs regualarly.


## Extending

There are several abstract interfaces which you can implement to support more message format, message queue and config management mechanism.

```
type Parser interface {
	Parse(bs []byte) model.Metric
}

type Inputer interface {
	Init(cfg *config.Config, taskName string, putFn func(msg model.InputMessage)) error
	Run(ctx context.Context)
	Stop() error
	CommitMessages(ctx context.Context, message *model.InputMessage) error
}

// RemoteConfManager can be implemented by many backends: Nacos, Consul, etcd, ZooKeeper...
type RemoteConfManager interface {
	Init(properties map[string]interface{}) error
	// Register this instance, and keep-alive via heartbeat.
	Register(ip string, port int) error
	Deregister(ip string, port int) error
	// GetInstances fetchs healthy instances.
	// Mature service-discovery solutions(Nacos, Consul etc.) have client side cache
	// so that frequent invoking of GetInstances() and GetGlobalConfig() don't harm.
	GetInstances() (instances []Instance, err error)
	// GetConfig fetchs the config. The manager shall not reference the returned Config object after call.
	GetConfig() (conf *Config, err error)
	// PublishConfig publishs the config. The manager shall not reference the passed Config object after call.
	PublishConfig(conf *Config) (err error)
}

```
