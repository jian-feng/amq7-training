# Networking Brokers

## Prerequisites
 
-   worksheet1

## Creating 2 Clustered Brokers

-   create broker 1

```code
(A_MQ_Install_Dir)/bin/artemis create  --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword  --max-hops 1 broker1
```

-   create broker 2

```code
(A_MQ_Install_Dir)/bin/artemis create  --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword  --max-hops 1 --port-offset 100 broker2
```

## config static discovery

By default, artemis will create clustered brokers with UDP-based dynamic discovery. We will change it to static discovery.

-   Add Connector used to discover broker 2

`<connector>` to `/broker1/etc/broker.xml`
```xml
   <!-- Connector used to discover broker 2 -->
   <connector name="broker2-connector">tcp://127.0.0.1:61716</connector>
```

-   Delete these 2 blocks:
    -    <broadcast-groups>  
    -    <discovery-groups>

-   Replace the line inside <cluster-connections> as following:

BEFORE:
```xml
<discovery-group-ref discovery-group-name="***" />
```

AFTER:
```xml
<static-connectors>
   <connector-ref>broker2-connector</connector-ref>
</static-connectors>
```



-   start both brokers
```code
./broker1/bin/artemis run
./broker2/bin/artemis run
```

After some initial negotiation you should see each broker log that a bridge has been created.
```console
AMQ221027: Bridge ClusterConnectionBridge@4445b19e [name=...] is connected
```

## clustering a queue

Both brokers need to be configured with the same anycast queue definition

-   stop both brokers

-   `broker[1,2]/etc/broker.xml` -> `<addresses>` Add an anycast queue configuration for both brokers
```xml
   <address name="exampleQueue">
      <anycast>
         <queue name="exampleQueue" />
      </anycast>
   </address>
```

-   start both brokers
```code
./broker1/bin/artemis-service start
./broker2/bin/artemis-service start
sleep 5
tail -n 3 ./broker1/log/artemis.log
tail -n 3 ./broker2/log/artemis.log
```

-   from the worksheet2 directory run 2 consumers connected to each broker
```code
mvn verify -PqueueReceiver1
mvn verify -PqueueReceiver2                                      
```

-   from the worksheet2 directory run a producer to send 10 messages
```code
mvn verify -PqueueSender
```

./broker1/bin/artemis queue purge --name exampleQueue
./broker2/bin/artemis queue purge --name exampleQueue

./broker1/bin/artemis-service restart
./broker2/bin/artemis-service restart

./broker1/bin/artemis queue stat --queueName exampleQueue
./broker2/bin/artemis queue stat --queueName exampleQueue



NB you will see each consumer receives 5 messages in a round robin fashion

> **TROUBLE-SHOOTING**
> AMQ Brokerのデータディレクトリのあるディスク容量が90パーセント以上使用されている場合、デフォルトの設定だと、AMQ Brokerがメッセージの受付をブロックします。クライアントがメッセージの送信に失敗します。
> クライアント側は、以下のようなメッセージが表示されます。
> WARN: AMQ212054: Destination address=exampleQueue is blocked. If the system is configured to block make sure you consume messages on this configuration.
> AMQ Brokerのログ(`broker[1,2]/log/artemis.log`)に次のようなメッセージが表示されます。
> AMQ222212: Disk Full! Blocking message production on address 'exampleQueue'. Clients will report blocked.
> 回避策は、`broker[1,2]\etc\broker.xml`の設定`<max-disk-usage>`を90から100に変更してください。


> AMQ212075: KQueue is not available, please add to the classpath or configure useKQueue=false to remove this warning
> `etc/broker.xml`
> <acceptor name="xxx">tcp://0.0.0.0:...?...;useKQueue=false</acceptor>
> <connector name="xxx">tcp://localhost:...?useKQueue=false</connector>



-   kill the consumers and run the producer again

```code
mvn verify -PqueueSender
```
-   now run the consumers again
```code
mvn verify -PqueueReceiver1
mvn verify -PqueueReceiver2                                      
```

NB you will see that only 1 consumer received the messages as they were load balanced on demand.



-   now stop the brokers and update the load balancing to `STRICT` in the `broker[1,2]/etc/broker.xml` file
```xml
<message-load-balancing>STRICT</message-load-balancing>
```

STRICT:  
  Forwards messages to all brokers that have a matching queue, whether or not the queue has an active consumer or a matching selector.

-   now re run the exercise and notice that both the consumers receive messages even when disconnected.

## clustering a topic

Both brokers need to be configured with the same multicast topic definition

-   Add an multicast topic configuration for both brokers
```xml
   <address name="exampleTopic">
      <multicast>
         <queue name="exampleTopic" />
      </multicast>
   </address>
```

-   from the worksheet2 directory run 2 consumers connected to each broker
```code
mvn verify -PtopicReceiver1
mvn verify -PtopicReceiver2                                      
```

-   from the worksheet2 directory run a producer to send 10 messages
```code
mvn verify -PtopicSender
```

NB you will see each consumer receives all 10 messages



