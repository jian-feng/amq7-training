# Broker Cluster


<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=3 orderedList=false} -->

<!-- code_chunk_output -->

- [Broker Cluster](#broker-cluster)
  - [前提条件](#%e5%89%8d%e6%8f%90%e6%9d%a1%e4%bb%b6)
  - [1. 2つBrokerの構成するクラスタを作成](#1-2%e3%81%a4broker%e3%81%ae%e6%a7%8b%e6%88%90%e3%81%99%e3%82%8b%e3%82%af%e3%83%a9%e3%82%b9%e3%82%bf%e3%82%92%e4%bd%9c%e6%88%90)
  - [2. Static Discoveryに設定](#2-static-discovery%e3%81%ab%e8%a8%ad%e5%ae%9a)
  - [3. Queueのクラスタを定義](#3-queue%e3%81%ae%e3%82%af%e3%83%a9%e3%82%b9%e3%82%bf%e3%82%92%e5%ae%9a%e7%be%a9)
  - [4. message-load-balancing=ON_DEMANDの動作確認](#4-message-load-balancingondemand%e3%81%ae%e5%8b%95%e4%bd%9c%e7%a2%ba%e8%aa%8d)
  - [5. message-load-balancing=STRICTの動作確認](#5-message-load-balancingstrict%e3%81%ae%e5%8b%95%e4%bd%9c%e7%a2%ba%e8%aa%8d)
  - [6. Topicのクラスタの設定とテスト](#6-topic%e3%81%ae%e3%82%af%e3%83%a9%e3%82%b9%e3%82%bf%e3%81%ae%e8%a8%ad%e5%ae%9a%e3%81%a8%e3%83%86%e3%82%b9%e3%83%88)

<!-- /code_chunk_output -->


## 前提条件
 
-   worksheet1の **Lab 1: AMQ Brokerのインストールと起動** が実施済みのこと

## 1. 2つBrokerの構成するクラスタを作成

1. broker1を作成します。
   以下のコマンドは`C:\redhat`の配下に、`broker1`というBrokerインスタンスを新規作成します。
   ```code
   (A_MQ_Install_Dir)\bin\artemis create  --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword  --max-hops 1 C:\redhat\broker1
   ```

2. broker2を作成します。
   以下のコマンドは`C:\redhat`の配下に、`broker2`というBrokerインスタンスを新規作成します。
   ```code
   (A_MQ_Install_Dir)\bin\artemis create  --user admin --password password --role admin --allow-anonymous y --clustered --host 127.0.0.1 --cluster-user clusterUser --cluster-password clusterPassword  --max-hops 1 --port-offset 100 C:\redhat\broker2
   ```
   `--port-offset 100` は、Brokerの使用したポートをすべて+100でずらすこと

## 2. Static Discoveryに設定

Broker Clusterは、初期値でUDP-base discoveryと設定されます。以下の手順でStatic Discoveryに変更します。

1. Broker1の設定ファイル`/broker1/etc/broker.xml`を開いて、
      1.1. Broker1からBroker2のCluster Connectorを`<connectors>`に追加します。
      > 既存の`<connector name="artemis">`は削除しないでください。
      ```xml
      <!-- Connector used to discover broker 2 -->
      <connector name="broker2-connector">tcp://127.0.0.1:61716</connector>
      ```

      1.2. `<broadcast-groups>` と `<discovery-groups>` のブロック削除します。

      1.3. `<cluster-connections>` の中の1行を以下のように書き換えます。
      BEFORE:
      ```xml
      <discovery-group-ref discovery-group-name="XXXXX" />
      ```
      AFTER:
      ```xml
      <static-connectors>
         <connector-ref>broker2-connector</connector-ref>
      </static-connectors>
      ```

2. Broker2の設定ファイル`/broker2/etc/broker.xml`を開いて、
      2.1. Broker2からBroker1のCluster Connectorを`<connectors>`に追加します。
      > 既存の`<connector name="artemis">`は削除しないでください。
      ```xml
      <!-- Connector used to discover broker 1 -->
      <connector name="broker1-connector">tcp://127.0.0.1:61616</connector>
      ```

      2.2. `<broadcast-groups>` と `<discovery-groups>` のブロック削除します。

      2.3. `<cluster-connections>` の中の1行を以下のように書き換えます。
      BEFORE:
      ```xml
      <discovery-group-ref discovery-group-name="XXXXX" />
      ```
      AFTER:
      ```xml
      <static-connectors>
         <connector-ref>broker1-connector</connector-ref>
      </static-connectors>
      ```


3. 2つのコマンドプロンプトを開いて、それそれでBrokerインスタントを起動します。
    ```
    C:\redhat\broker1\bin\artemis run
    C:\redhat\broker2\bin\artemis run
    ```

4. Brokerインスタンスのログ（コマンドプロンプト上）では、Bridgeが接続された旨のメッセージが表示されることを確認できます。
```console
AMQ221027: Bridge ClusterConnectionBridge@4445b19e [name=...] is connected
```

## 3. Queueのクラスタを定義

2つのBrokerともにanycastのQueueを登録します。
  `broker[1,2]/etc/broker.xml` の `<addresses>` ブロックに、以下のようにQueueを追記します。
   ```xml
   <address name="exampleQueue">
      <anycast>
         <queue name="exampleQueue" />
      </anycast>
   </address>
   ```

## 4. message-load-balancing=ON_DEMANDの動作確認

1. 2つのコマンドプロンプトを開いて、それぞれのターミナルにて、練習用コンテンツのworksheet2ディレクトリから、exampleQueueからメッセージ受信を開始します。
    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PqueueReceiver1
    ```

    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PqueueReceiver2
    ```

2. さらに1つのコマンドプロンプトを開いて、練習用コンテンツのworksheet2ディレクトリから、exampleQueueへメッセージ送信します。
   ```code
   cd C:\redhat\amq7-training-master\worksheet2
   mvn verify -PqueueSender
   ```

3. queueReceiver1のターミナルでは、5件受信したことを確認できます。
   ```
   Awaiting message
   message = this is the 1st message
   Awaiting message
   message = this is the 3rd message
   Awaiting message
   message = this is the 5th message
   Awaiting message
   message = this is the 7th message
   Awaiting message
   message = this is the 9th message
   Awaiting message
   ```

4. queueReceiver2のターミナルでも、5件受信したことを確認できます。
   ```
   Awaiting message
   message = this is the 2nd message
   Awaiting message
   message = this is the 4th message
   Awaiting message
   message = this is the 6th message
   Awaiting message
   message = this is the 8th message
   Awaiting message
   message = this is the 10th message
   Awaiting message
   ```

2つのBrokerにそれぞれに受信者(queueReceiver[1,2])がいって、queueSenderによって送信された10件のメッセージが、ラウンドロビン方式で2つのBrokerに分配され、queueReceiver[1,2]に受信したことを確認できました。

1. 前に起動したqueueReceiver[1,2]を停止して、再度queueSenderを実行します。
   ```code
   cd C:\redhat\amq7-training-master\worksheet2
   mvn verify -PqueueSender
   ```

2. 前回同様に、2つのコマンドプロンプトを開いて、それぞれのターミナルにて、exampleQueueからメッセージ受信を開始します。
    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PqueueReceiver1
    ```

    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PqueueReceiver2
    ```

3. queueReceiver1のターミナルでは、10件受信したことを確認できます。
   ```
   Awaiting message
   message = this is the 1st message
   Awaiting message
   message = this is the 10th message
   Awaiting message
   ```

4. queueReceiver2のターミナルは0件受信したことを確認できます。

2つのBrokerにそれぞれに受信者(queueReceiver[1,2])がいなかったため、10件のメッセージが、On-demand方式で先に接続されたBroker1に分配され、後ほど、queueReceiver1だけが受信したことを確認できました。

## 5. message-load-balancing=STRICTの動作確認

> `<cluster-connections>`設定変更は、Brokerの再起動が必要のため、
> 一旦、2つのBrokerをすべて停止して設定変更します。

1. Brokerインスタントを停止します。
   先程Broker起動したコンソールにて`Ctrl + C`を押すか、別のコマンドプロンプトで以下のコマンドを実行して、Brokerインスタンスを停止します。
    ```
    C:\redhat\broker1\bin\artemis stop
    C:\redhat\broker2\bin\artemis stop
    ```


2. Broker[1,2]の設定ファイル`etc/broker.xml`を開いて、message-load-balancingをON-DEMANDから`STRICT`に変更します。
   ```xml
   <message-load-balancing>STRICT</message-load-balancing>
   ```

   - `STRICT` とは、Queueが存在するBrokerがいれば、そのBrokerへメッセージを転送します。Broker上に受信者有無はチェックしません。また、複数Brokerがいれば、ラウンドロビン方式で分配します。  

3. 2つのコマンドプロンプトを開いて、それそれでBrokerインスタントを起動します。
    ```
    C:\redhat\broker1\bin\artemis run
    C:\redhat\broker2\bin\artemis run
    ```

4. 前に起動したqueueReceiver[1,2]が停止した状態で、queueSenderを実行します。
   ```code
   cd C:\redhat\amq7-training-master\worksheet2
   mvn verify -PqueueSender
   ```

5. 2つのコマンドプロンプトを開いて、それぞれのターミナルにて、exampleQueueからメッセージ受信を開始します。
    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PqueueReceiver1
    ```

    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PqueueReceiver2
    ```

6. queueReceiver[1,2]のターミナルで、それぞれ5件受信したことを確認できます。

10件のメッセージが、ラウンドロビン方式でBroker[1,2]に分配され、後ほど、queueReceiver[1,2]によって受信したことを確認できました。


## 6. Topicのクラスタの設定とテスト

1. 2つのBrokerともにmulticastのQueueを登録します。
  `broker[1,2]/etc/broker.xml` の `<addresses>` ブロックに、以下のようにQueueを追記します。
   ```xml
   <address name="exampleTopic">
      <multicast/>
   </address>
   ```

2. 2つのコマンドプロンプトを開いて、それぞれのターミナルにて、exampleTopicからメッセージ受信を開始します。
    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PtopicReceiver1
    ```

    ```
    cd C:\redhat\amq7-training-master\worksheet2
    mvn verify -PtopicReceiver2
    ```

3. もう1つのコマンドプロンプトにてtopicSenderを実行します。
   ```code
   cd C:\redhat\amq7-training-master\worksheet2
   mvn verify -PtopicSender
   ```

4. topicReceiver[1,2]のターミナルで、それぞれ10件受信したことを確認できます。


