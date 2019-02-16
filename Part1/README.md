# KafkaSASL_SSLAuthExample

**Part 1<br/>
Adjust brokers to use SASL_SSL instead of SSL**

In this part, you will migrate your inter broker communication from SSL on port 9093 to SASL_SSL on port 9094.

Here are the steps we will take:
- Add SASL_SSL to your broker listeners
- Update the inter broker protocol to specify SASL_SSL
- And add the requisite GSSAPI parameters

GSSAPI is the Kerberos mechanism of SASL.

This ubuntu identifier must match the kerberos principal from the kafka service keytab.

On your Kerberos server, if you do not have a wide open security group (which is wise), you should add UDP port as an internal access port. The reason you do this is because we previously allowed TCP on port 88. This is sufficient to do a manual kinit to grab a ticket. However, since we are going to use the Kafka server we also have to allow UDP port 88.

```
sudo vi ../etc/kafka/server.properties

listeners=PLAINTEXT://<your kafka server>:9092,SSL://<your kafka server>:9093,SASL_SSL://<your kafka server>:9094

advertised.listeners=PLAINTEXT://<your kafka server>:9092,SSL://<your kafka server>:9093,SASL_SSL://<your kafka server>:9094

security.inter.broker.protocol=SASL_SSL

sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.service.name=ubuntu
```

Create the JaaS configuration file for the Kafka broker.

```
sudo vi ../etc/kafka/kafka_server_jaas.conf

KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/tmp/kafka.service.keytab"
    principal="ubuntu/<your kafka server>@KAFKA.SECURE";
};
```

Again, in our  example here we have ubuntu server running Kafka (running as user ubuntu) and krb5-user and another server, the CentOS server, running the krb5-server.

On the Kerberos server machine, let's create the principal and keytab as such:

```
sudo kadmin.local -q "add_principal -randkey ubuntu/<your kafka server>@KAFKA.SECURE"

sudo kadmin.local -q "xst -kt /tmp/kafka.service.keytab ubuntu/<your kafka server>@KAFKA.SECURE"
```

Copy those keytabs to local laptop and up to the krb5-user/Kafka server so that when we start Kafka (and pass in the proper JaaS file upon Kafka startup), it will have access to the right keytab. Of course, you can bypass your laptop and go straight from server to server.

```
scp -i ~<your pem file>.pem centos@<your kerberos server>:/tmp/*.keytab /tmp/

scp -i ~<your pem file>.pem /tmp/*.keytab ubuntu@<your kafka server>:/tmp
```

On your kafka server, grab a ticket!

```
kinit -kt /tmp/kafka.service.keytab ubuntu/<your kafka server>
```

Verify it was ok with expiration.

```
klist
```

Start Kafka.

Now, launch Kafka using the right environment variable. If you were using systemd, you would include the following in your systemd script.
```
Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/home/ubuntu/confluent-5.1.1/etc/kafka/kafka_server_jaas.conf"
```

If you are running it manually like this example, you can do something like this:

```
./confluent start zookeeper

KAFKA_OPTS="-Djava.security.auth.login.config=/home/ubuntu/confluent-5.1.1/etc/kafka/kafka_server_jaas.conf" ./confluent start kafka

./confluent start schema-registry
./confluent start kafka-rest
./confluent start connect
./confluent start ksql-server
./confluent start control-center
```

Verify everything is ok.

Note that upon startup, there are a number of kerberos issues that can occur during this configuration and testing, but here are a couple of common ones to watch out for in your broker log files upon startup.

If you specify the wrong principal in your jaas config, the error may resemble:

```
org.apache.kafka.common.KafkaException: javax.security.auth.login.LoginException: Could not login: the client is being asked for a password, but the Kafka client code does not currently support obtaining a password from the user. not available to garner  authentication information from the user
```

If you specify the wrong principal in your broker server.properties file, the error may resemble:

```
Caused by: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Server not found in Kerberos database (7) - LOOKING_UP_SERVER)]
```
