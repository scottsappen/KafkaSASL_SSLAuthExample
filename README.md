# KafkaSASL_SSLAuthExample
This is a build upon the KafkaSSLAuthExample repo where we get SASL_SSL working.

It's a simple example of getting SASL_SSL configured.

Keep in mind there are several SASL implementations available for kafka and in this example we will use the kerberos implementation.

**Prereqs**

Pre-reqs:
- See repo MITKerberosExample if you want to run your own MIT kerberos - https://github.com/scottsappen/MITKerberosExample
- See example if you need to setup SSL - https://github.com/scottsappen/KafkaSSLAuthExample

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

You can use whatever boxes and O/S you want.

In this example, we will re-use the Ubuntu box in AWS for Kafka from the repo KafkaSSLAuthExample. However, something like this would be fine:<br/>
x4.large, AWS Marketplace (search for Ubuntu), Ubuntu 16.04 LTS - Xenial (HVM). For your Kafka client, you can use the same Ubuntu box and pretend that it is not only a server, but also a client. In this example, we will stick to using a separate laptop that has kafka and the various networking tools installed.

**What we will do**

See the subdirectories in this repo for the details of each part:

- Part 1 - Adjust brokers to use SASL_SSL instead of SSL
- Part 2 - Using clients with kerberos

That's it. When you're done, I hope you will have enjoyed this walkthrough and learned something new.
