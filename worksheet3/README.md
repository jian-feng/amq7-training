# High Availability

## Prerequisites

-   worksheet1 and worksheet2

## Shared Store

### Creating a Shared Store Pair

-   create live broker

```code
(A_MQ_Install_Dir)/bin/artemis create --shared-store --failover-on-shutdown  --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword --max-hops 1 liveBroker
```
-   create a backup broker

```code
 (A_MQ_Install_Dir)/bin/artemis create --shared-store --failover-on-shutdown --slave --data ../liveBroker/data --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword --max-hops 1 --port-offset 100 backupBroker
```

### config static discovery

By default, artemis will create clustered brokers with UDP-based dynamic discovery. We will change it to static discovery.

-   Add Connector used to discover backupBroker

`<connector>` to `/broker1/etc/broker.xml`
```xml
   <!-- Connector used to discover backupBroker -->
   <connector name="backupBroker-connector">tcp://127.0.0.1:61716</connector>
```

-   Delete these 2 blocks:
    -    `<broadcast-groups>`  
    -    `<discovery-groups>`

-   Replace the line inside `<cluster-connections>` as following:

BEFORE:
```xml
<discovery-group-ref discovery-group-name="***" />
```

AFTER:
```xml
<static-connectors>
   <connector-ref>backupBroker-connector</connector-ref>
</static-connectors>
```

### Config Queue for test

-   add a queue to each broker
```xml
   <address name="exampleQueue">
      <anycast>
         <queue name="exampleQueue" />
      </anycast>
   </address>
```

### Run test for live-backup failover

-   start both brokers
```code
liveBroker/bin/artemis run
backupBroker/bin/artemis run
```
You should see the backup broker announce itself as a backup in the logs.
```
AMQ221109: Apache ActiveMQ Artemis Backup Server version 2.10.0.redhat-00004 [2538ddcb-3db5-11ea-80f3-52e42774a09e] started, waiting live to fail before it gets active
```

-   Now kill the live broker

The backup should start as a live


-   Now restart the live broker

Since `<failover-on-shutdown>` is set to true you should see the backup automatically shut down and the live restart

- Now configure the clients to use ha, change the connection factory in the jndi.properties to be
```code
connectionFactory.ConnectionFactory=(tcp://localhost:61616?ha=true&retryInterval=1000&retryIntervalMultiplier=1.0&reconnectAttempts=-1,tcp://localhost:61716?ha=true&retryInterval=1000&retryIntervalMultiplier=1.0&reconnectAttempts=-1)
```

Notice we have configured both brokers in the connection factory but could have used udp

-   from the worksheet3 directory start a queue receiver

```code
mvn verify -PqueueReceiver
```

-   from the worksheet3 directory start the sender

```code
mvn verify -PqueueSender
```

you will see messages produced and consumed

-   now kill the live broker

You will see the backup broker take over and then clients continue to send and receive messages with maybe a warning.

-   now kill the backup

You will see clients with connection failure as before.

```
AMQ212037: Connection failure to /127.0.0.1:61616 has been detected: AMQ219015: The connection was disconnected because of server shutdown [code=DISCONNECTED]
```

-   and start backup

Once the backup is restarted the clients will continue

-   now restart the live broker

The live will resume its responsibilities and clients will carry on

## [NOT CONFIRMED] Creating a Shared Nothing Pair (replicated)

> **A Note on shared nothing high availability**  
> Shared nothing HA is only supported in a single local data center preferably on the same network subnet. Shared nothing HA is very sensitive to network latency and consistency therefore shared nothing HA configurations where the master is in one data center and the slave is in another are specifically not supported.

-   create a live broker

```code
$ARTEMIS_HOME/bin/artemis create --replicated --failover-on-shutdown  --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword  --max-hops 1 repLiveBroker
```

-   create a backup broker    

```code
$ARTEMIS_HOME/bin/artemis create --replicated --failover-on-shutdown --slave --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword  --max-hops 1 --port-offset 100 repBackupBroker
```
-   add a queue to each broker.xml
```xml
<address name="exampleQueue">
   <anycast>
      <queue name="exampleQueue" />
   </anycast>
</address>
```

-   start both brokers

 ```code
 (repLiveBrokerHome)/bin/artemis run
 (repBackupBrokerHome)/bin/artemis run
 ```
 You should see the backup broker announce itself as a backup in the logs and the journal being replicated.

 -   Now kill the live broker


 The backup should start as a live

 -  Before we restart the live broker we need to add some configuration to the live broker, so update the broker.xml and update

 ```xml
 <ha-policy>
  <replication>
     <master>
        <check-for-live-server>true</check-for-live-server>
     </master>
  </replication>
</ha-policy>
 ```

 Without this the live broker would just start without checking for a backup broker that has failed over so always configure this

 -   Now restart the live broker

you will see the live broker start and replicate back from the backup but notice it doesnt start

-   So manually kill the backup broker

You will see the live take over

-   now to automate failback update the backups broker.xml with

```xml
<ha-policy>
 <replication>
    <slave>
       <allow-failback>true</allow-failback>
    </slave>
 </replication>
</ha-policy>
```

-   now restart the backup and wait for replication to start
-   Kill the live broker
-   restart the live

you will now see the live broker replicate back then take over as live


-   from the worksheet3 directory start a queue receiver

```code
mvn verify -PqueueReceiver
```

-   from the worksheet3 directory start the sender

```code
mvn verify -PqueueSender
```

- play around killing the brokers
