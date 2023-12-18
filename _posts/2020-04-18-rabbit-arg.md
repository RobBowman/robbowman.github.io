---
title: RabbitMQ Inequivalent Arg Error
category: RabbitMQ
tags:
    - RabbitMQ
---
# Rabbit MQ Inequivalent Arg Error
I have an Azure function that's making use of the new(ish) RabbitMQ binding to collect messages. This binding (https://github.com/Azure/azure-functions-rabbitmq-extension) works well but at time of writing has a few limitations; one of which is that it cannot collect from "durable" queues.

Use of 'transient' queues in my scenario is ok, so when I created my new queue through the RabbitMQ UI I simply set the "Durability" option to "Transient".

This worked well for a few days, then I noticed that the Azure function had stopped collecting messages from the queue. I checked the host logs using Kudu and found the following error:

>"PRECONDITION_FAILED - inequivalent arg 'durable' for queue 'avaloq_client' in vhost '/': received 'false' but current is 'true'"

The "current is'true'" bit had me very confused! I was sure that the durability of the queue had not been updated by anyone so how did this change?

To be honest I'm still not sure I know the answer but I now suspect a RabbitMQ policy may by the problem.

On the Admin\Policies page of the RabbitMQ UI, I found the following policy in place:

![Rabbit Policy](/images/rabbit-arg/1.png)

So, it seems this policy named "HA" sets the ha-mode for all queues. My guess is that this impacts the "durable" setting for my queue.

There are many other queues on the Rabbit server, and this policy could be good for them. So, in order to keep everyone happy I haven't deleted the policy but just update the regular expression so that the policy doesn't get applied to my queue.

I hope this will fix the problem but I guess I won't know for a few days!