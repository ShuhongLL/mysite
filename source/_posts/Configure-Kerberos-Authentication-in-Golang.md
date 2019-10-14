---
title: Configure Kerberos Authentication in Golang
date: 2019-10-10 19:54:37
tags: [Kafka, Golang, Kerberos]
photos: ["../images/kerberos.png"]
---

This is a short guide about how to configure kerberos authentication on the client side (server side already been configured) using Golang
<!-- more -->

## Breif introduction of Kerberos

Kerberos is a protocol developed by MIT scientists with the name introduced by the [three-head dog](https://en.wikipedia.org/wiki/Cerberus) from myth. It is well-spreaded as a general authorization technology and used across multiple platforms.

The basic steps to get authentication are:
1. A Client requests an authentication ticket(TGT) with credentials from the Key Distribution Center(KDC)
2. KDC verifies the credentials and returns an encrypted TGT
3. Client saves TGT and sends the encrypted TGT to the Ticket Granting Service(TGS)
4. KDC verifies TGT and notifies TGS, then TGS returns a valid session token to the client
5. Client uses the token to access a specific server
6. ( If TGT expires, client will request for a new one by calling ` kinit ` ) 
</br>

## Go Client

There are multipe clients available for Golang, and you can refer to [here](https://cwiki.apache.org/confluence/display/KAFKA/Clients) to have a breif look. Usually we intent to choose libraries which are purely coded in the sme programming language to avoid importing unnecessary dependencies. Unfortunately, by the time I wrote this article, I haven't found out a library which is purely written by Go, therefore I choose a cgo library supported by [Confluent](https://github.com/confluentinc/confluent-kafka-go), this library refers to a C library [librdkafka](https://github.com/edenhill/librdkafka).
</br>

## Environment

For the reason that I couldn't refresh my keytab on MacOS (most likely because macos disable UDP connection by default and hence cannot recognize kdc in my realms), I switched to linux (for dockerfile, linux is also a good choice :P).

The Go version is required to be at least `1.12`

Firstly, since our client refering to `librdkafka`, we need to install librdkafka:
```
wget -qO - https://packages.confluent.io/deb/5.3/archive.key | apt-key add - && \
add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/5.3 stable main"
```

Then install:
```
apt-get install -y librdkafka-dev
```

I am going to use `GSSAPI` as `SASL` authentication and kerberos `krb` configuration hence need to import a few more tools:
```
apt-get install -y libsasl2-modules-gssapi-mit libsasl2-dev
apt-get install -yqq krb5-user libpam-krb5
```

`libsasl2-modules-gssapi-mit` and `libsasl2-dev` are specified to GSSAPI authentication, if you are using other mechanism (for example PLAIN or SCRAM-SHA-256), please refer to corresponding tools.


And double check if you have `ca-certificates` installed (Not sure why, but without ca-certificates, krb configuration cannot be set)
</br>

## Create a Client in Go

```go
import kafka "github.com/confluentinc/confluent-kafka-go/kafka"

client, err := kafka.NewConsumer(&kafka.ConfigMap{
    // Avoid connecting to IPv6 brokers:
    // when using localhost brokers on OSX, since the OSX resolver
    // will return the IPv6 addresses first.
    // You typically don't need to specify this configuration property.
    // "broker.address.family":      "v4",
    "bootstrap.servers":          "[Server host:port]",
    "group.id":                   "[Group id]",
    "security.protocol":          "SASL_PLAINTEXT",
    "session.timeout.ms":         6000,
    "sasl.mechanism":             "GSSAPI",
    "auto.offset.reset":          "earliest",
    "sasl.kerberos.service.name": "[Service name]",
    "sasl.kerberos.keytab":       "[Key tab location]",
    "sasl.kerberos.principal":    "[Principal]",
    "sasl.kerberos.kinit.cmd":    "kinit -R -t \"%{sasl.kerberos.keytab}\" -k %{sasl.kerberos.principal}",      
})
```

Then export [JAAS configurations](https://docs.confluent.io/current/kafka/authentication_sasl/index.html)
```
export KRB5_CONFIG="/usr/local/kafka/conf/krb5/krb5.conf"
export KAFKA_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/conf/kafka/kafka_client_jaas.conf-Djava.security.krb5.conf=/usr/local/kafka/conf/krb5/krb5.conf -Dsun.security.krb5.debug=true"
```

For here, I use `keytab` to authorize which need to configured on kafka server side. Initially, we can use `username` and `password` to authorize.

```
    "sasl.username": username,
    "sasl.password": password,
```

Other configuration fields can be referred to the [offical documentaion](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md)
</br>

## Dockerfile

Well, if you don't wanna spend time on the above configurations, here is the dockerfile :D
```
# refer to a cgo library maintained by Confluent: https://github.com/confluentinc/confluent-kafka-go
# which requires a C dependency librdkafka-dev: https://github.com/edenhill/librdkafka
# The C dependency librdkafka-dev is curretly not available for other linux version except for ubuntu/debian.
FROM ubuntu

ENV http_proxy=
ENV https_proxy=

ENV DEBIAN_FRONTEND=noninteractive

# Install the C lib for kafka
RUN apt-get update && \
    apt-get install -y --no-install-recommends apt-utils wget gnupg software-properties-common && \
    apt-get install -y apt-transport-https ca-certificates git curl openssl libsasl2-modules-gssapi-mit libsasl2-dev && \
    apt-get install -yqq krb5-user libpam-krb5 && \
# import source repository from confluent, check the latest version on https://docs.confluent.io/current/installation/installing_cp/deb-ubuntu.html#get-the-software
    wget -qO - https://packages.confluent.io/deb/5.3/archive.key | apt-key add - && \
    add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/5.3 stable main" && \
# import the librdkafka-dev from confluent source repository
# confluent-kafka-go always requires the latest librdkafka-dev library
# If go build fail below because of the mismatch of confluent-kafka-go and librdkafka-dev,
# please check the latest source repositary on https://docs.confluent.io/current/installation/installing_cp/deb-ubuntu.html#get-the-software
    apt-get install -y librdkafka-dev && \
# Install Go
    add-apt-repository ppa:longsleep/golang-backports && \
    apt-get install -y golang-1.12-go

# build the library
WORKDIR /src

ADD . /src

RUN GOPATH=/go GOOS=linux /usr/lib/go-1.12/bin/go build -o app && \
    mv /src/app /usr/local/bin

ENV http_proxy ''
ENV https_proxy ''

EXPOSE 8000

ENTRYPOINT ["/usr/local/bin/app"]

```
