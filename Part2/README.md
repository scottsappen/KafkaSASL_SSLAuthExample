# KafkaSASL_SSLAuthExample

**Part 2<br/>
Using cients with kerberos**

In this part, you will use the console producer and consumer using kerberos with Kafka.

Here are the steps we will take:
- Prepare the requisite property files
- Run some tests

Let's create the requisite client files. Providing the trusstore is because you are using SASL on top of SSL authentication. The sasl.kerberos.service.name must match your specified user in your broker server properties.

```
sudo vi /tmp/kafka_client_jaas.conf

KafkaClient {
 com.sun.security.auth.module.Krb5LoginModule required
 useTicketCache=true;
 };

sudo vi /tmp/kafka_client_kerberos.properties

security.protocol=SASL_SSL
sasl.kerberos.service.name=ubuntu
ssl.truststore.location=/home/ubuntu/confluent-5.1.1/ssl/kafka.client.truststore.jks
ssl.truststore.password=confluent
```

Now let's also make sure our keytab is available to use and we have a corresponding ticket in the cache from a kinit.

Let's use the reader keytab we made available to ourselves from the MITKerberosExample walkthrough. Issue a klist to see if you have a valid ticket in the cache.

```
klist
```

You should see something like this:

```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: reader@KAFKA.SECURE

Valid starting       Expires              Service principal
02/16/2019 18:24:56  02/17/2019 18:24:56  krbtgt/KAFKA.SECURE@KAFKA.SECURE
```

(Optional) If you do not (or if you need to do a kinit to get a new ticket), you can read up on that walkthrough again for help in issuing these type of commands for keytabs on the kerberos server. You would of course upload the corresponding reader keytab from the kerberos server to the kafka server or the server where you are running a producer/consumer test.

```
sudo kadmin.local -q "add_principal -randkey reader@KAFKA.SECURE"

sudo kadmin.local -q "xst -kt /tmp/reader.user.keytab reader@KAFKA.SECURE"

kinit -kt /tmp/reader.user.keytab reader
```

Now let's run a sample producer application using the reader keytab.

```
export KAFKA_OPTS="-Djava.security.auth.login.config=/tmp/kafka_client_jaas.conf"

./kafka-console-producer --broker-list <your kafka server>:9094 --topic kafka-security-topic --producer.config /tmp/kafka_client_kerberos.properties
```

Or in one line, you can accomplish the same thing.

```
KAFKA_OPTS="-Djava.security.auth.login.config=/tmp/kafka_client_jaas.conf" ./kafka-console-producer --broker-list <your kafka server>:9094 --topic kafka-security-topic --producer.config /tmp/kafka_client_kerberos.properties
```

Now let's run a sample consumer application using the same reader keytab.

```
KAFKA_OPTS="-Djava.security.auth.login.config=/tmp/kafka_client_jaas.conf" ./kafka-console-consumer --bootstrap-server <your kafka server>:9094 --topic kafka-security-topic --consumer.config /tmp/kafka_client_kerberos.properties
```

Success!

Now let's see what you can expect if you do not have valid credentials.

```
kdestroy

klist
```

You should see something that resembles this:

```
klist: Credentials cache file '/tmp/krb5cc_1000' not found
```

Now let's run a sample consumer application using the same reader keytab.

```
KAFKA_OPTS="-Djava.security.auth.login.config=/tmp/kafka_client_jaas.conf" ./kafka-console-consumer --bootstrap-server <your kafka server>:9094 --topic kafka-security-topic --consumer.config /tmp/kafka_client_kerberos.properties
```

You should see an error that resembles this (the error is a bit misleading or non-helpful, but good to see):

```
[2019-02-16 18:47:26,116] ERROR Unknown error when running consumer:  (kafka.tools.ConsoleConsumer$)
org.apache.kafka.common.KafkaException: Failed to construct kafka consumer
	at org.apache.kafka.clients.consumer.KafkaConsumer.<init>(KafkaConsumer.java:805)
	at org.apache.kafka.clients.consumer.KafkaConsumer.<init>(KafkaConsumer.java:652)
	at kafka.tools.ConsoleConsumer$.run(ConsoleConsumer.scala:67)
	at kafka.tools.ConsoleConsumer$.main(ConsoleConsumer.scala:54)
	at kafka.tools.ConsoleConsumer.main(ConsoleConsumer.scala)
Caused by: org.apache.kafka.common.KafkaException: javax.security.auth.login.LoginException: Could not login: the client is being asked for a password, but the Kafka client code does not currently support obtaining a password from the user. not available to garner  authentication information from the user
	at org.apache.kafka.common.network.SaslChannelBuilder.configure(SaslChannelBuilder.java:155)
	at org.apache.kafka.common.network.ChannelBuilders.create(ChannelBuilders.java:140)
	at org.apache.kafka.common.network.ChannelBuilders.clientChannelBuilder(ChannelBuilders.java:65)
	at org.apache.kafka.clients.ClientUtils.createChannelBuilder(ClientUtils.java:108)
	at org.apache.kafka.clients.consumer.KafkaConsumer.<init>(KafkaConsumer.java:718)
	... 4 more
Caused by: javax.security.auth.login.LoginException: Could not login: the client is being asked for a password, but the Kafka client code does not currently support obtaining a password from the user. not available to garner  authentication information from the user
	at com.sun.security.auth.module.Krb5LoginModule.promptForPass(Krb5LoginModule.java:940)
	at com.sun.security.auth.module.Krb5LoginModule.attemptAuthentication(Krb5LoginModule.java:760)
	at com.sun.security.auth.module.Krb5LoginModule.login(Krb5LoginModule.java:617)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at javax.security.auth.login.LoginContext.invoke(LoginContext.java:755)
	at javax.security.auth.login.LoginContext.access$000(LoginContext.java:195)
	at javax.security.auth.login.LoginContext$4.run(LoginContext.java:682)
	at javax.security.auth.login.LoginContext$4.run(LoginContext.java:680)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.login.LoginContext.invokePriv(LoginContext.java:680)
	at javax.security.auth.login.LoginContext.login(LoginContext.java:587)
	at org.apache.kafka.common.security.authenticator.AbstractLogin.login(AbstractLogin.java:60)
	at org.apache.kafka.common.security.kerberos.KerberosLogin.login(KerberosLogin.java:103)
	at org.apache.kafka.common.security.authenticator.LoginManager.<init>(LoginManager.java:61)
	at org.apache.kafka.common.security.authenticator.LoginManager.acquireLoginManager(LoginManager.java:111)
	at org.apache.kafka.common.network.SaslChannelBuilder.configure(SaslChannelBuilder.java:144)
	... 8 more
```
