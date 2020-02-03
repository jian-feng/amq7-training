# Getting started


<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=2 orderedList=false} -->

<!-- code_chunk_output -->

- [練習用PCスペック:](#練習用pcスペック)
- [事前準備](#事前準備)
- [Lab 1: AMQ Brokerのインストールと起動](#lab-1-amq-brokerのインストールと起動)
- [Lab 2: Point-to-Pointの送受信(Queue)](#lab-2-point-to-pointの送受信queue)
- [Lab 3: Publish-Subscribeの送受信(Topic)](#lab-3-publish-subscribeの送受信topic)
- [TROUBLE-SHOOTING](#trouble-shooting)

<!-- /code_chunk_output -->

## 練習用PCスペック:

* メモリ4GB(推奨8GB)
  * 同時起動しない場合4GBでもOK
  * 8GBとは開発用IDEとAMQ/Fuseサーバを同時に起動する場合
* ハードディスク10GB空き(推奨10GB以上)
  * 最低10GBですが余裕をもってください
* インターネットアクセス
  * Mavenを使うのでインターネットにアクセスします

## 事前準備

### 1. OpenJDK 1.8 (JREではなくJDKです)
  - **Windowsの場合**
    下記リンク先からダウンロードしてインストールしてください。
    DL: https://developers.redhat.com/products/openjdk/download
    File: OpenJDK 8 Windows 64­-bit
    `jdk-8u232-x64 ZIP` または `jdk-8u232-x64 MSI`

  - **RHEL/Fedora/CentOSの場合**
    下記RPMをインストールしてください。
    `yum install java-1.8.0-openjdk-devel`

  - **Mac OSの場合**
    Oracle JDKをインストールしてください。
    http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

### 2. Apache Maven 3.6.3(最新版)
  - **Windowsの場合**
  インストール手順は下記をご参照ください。
  (mvnをPATHに通してください)
  https://qiita.com/Junichi_M_/items/20daee936cd0c03c3115

### 3. LibAIO (RHEL/Fedora/CentOSのみ)
  - **RHEL/Fedora/CentOSの場合**
    下記RPMをインストールしてください。
    `yum install libaio`

### 4. AMQ Brokerのダウンロード
  下記のリンク先からダウンロードして保存してください。
  > 注意: 保存先はスペースなしのディレクトリにしてください。
  インストールは、ハンズオン当日の練習で行います。
  DL: https://developers.redhat.com/products/amq/download
  File: `AMQ Broker - Red Hat AMQ Broker 7.5.0`

### 5. 練習用コンテンツ

  - Gitがインストールされている場合は、`git clone`してください。
    `git clone https://github.com/jian-feng/amq7-training.git`
    
  - Gitがインストールされてない場合は、ダウンロード・解凍してください。
    下記のリンク先のZipをダウンロードして解凍してください。
    https://github.com/jian-feng/amq7-training/archive/master.zip
    > 注意: 解凍先はスペースなしのディレクトリにしてください。

## Lab 1: AMQ Brokerのインストールと起動

1. 事前にダウンロード済みAMQ Brokerのインストールファイルをローカルディスクに解凍します。
   例えば、`c:\redhat` に解凍すると、artemisスクリプトは`c:\redhat\amq-broker-7.5.0\bin\artemis.cmd` になります。
  > 注意: 解凍先はスペースなしのディレクトリにしてください。
  > 以降に`A_MQ_Install_Dir`とは`c:\redhat\amq-broker-7.5.0`で読み替えてください。
  
2. Brokerインスタントを作成します。
    以下のコマンドは`C:\redhat`の配下に、`broker0`というBrokerインスタンスを新規作成します。また、adminユーザも登録します。
   ```
   (A_MQ_Install_Dir)\bin\artemis create  --user admin --password password --role admin --allow-anonymous y C:\redhat\broker0
   ```

3. Brokerインスタントを起動します。
    ```
    C:\redhat\broker0\bin\artemis run
    ```

4. Brokerの管理コンソールへアクセスします。
   ブラウザで下記URLを開いてください。
   http://localhost:8161/console
    ユーザ:    `admin`
    パスワード: `password`

5. Brokerインスタントを停止します。
   上記３の実行したコンソールにて`Ctrl + C`を押すか、別のコマンドプロンプトで以下のコマンドを実行して、Brokerインスタンスを停止します。
    ```
    C:\redhat\broker0\bin\artemis stop
    ```

## Lab 2: Point-to-Pointの送受信(Queue)

1. AddressとQueueを登録します。
   `C:\redhat\broker0\etc\broker.xml` を開いて、`<addresses>`ブロックの中に、以下のXML設定内容を追加して保存します。
    ```xml 
    <address name="exampleQueue">
      <anycast>
        <queue name="exampleQueue" />
      </anycast>
    </address>
    ```

2. Brokerインスタントを起動します。
    ```
    C:\redhat\broker0\bin\artemis run
    ```

3. 事前にダウンロードした練習用コンテンツのworksheet1ディレクトリから、以下のコマンドを実行して、メッセージをexampleQueueへ送信します。
    ```
    cd C:\redhat\amq7-training-master\worksheet1
    mvn verify -PqueueSender
    ```

4. 事前にダウンロードした練習用コンテンツのworksheet1ディレクトリから、以下のコマンドを実行して、exampleQueueからメッセージを受信します。
    ```
    cd C:\redhat\amq7-training-master\worksheet1
    mvn verify -PqueueReceiver
    ```

5. Brokerの管理コンソールへアクセスします。
   http://localhost:8161/console
   - ユーザ:    `admin`
   -  パスワード: `password`

    addresses -> exampleQueue -> queues -> "anycast" -> exampleQueueのAtrributesを表示して、`Messages added` と `Messages acknowledged`がそれぞれ"2"となっていることを確認します。
     - `Messages added` とは、Queueに追加されたメッセージ数
     - `Messages acknowledged` とは、Queueから取出したしたメッセージ数

## Lab 3: Publish-Subscribeの送受信(Topic)

1. AddressとQueueを登録します。
   `C:\redhat\broker0\etc\broker.xml` を開いて、`<addresses>`ブロックの中に、以下のXML設定内容を追加して保存します。
    ```xml 
    <address name="exampleTopic">
      <multicast/>
    </address>
    ```

2. worksheet1ディレクトリから、以下のコマンドを実行して、exampleTopicから受信待ちの開始します。
    ```
    cd C:\redhat\amq7-training-master\worksheet1
    mvn verify -PtopicReceiver
    ```

3. Brokerの管理コンソールへアクセスします。
   http://localhost:8161/console
   - ユーザ:    `admin`
   -  パスワード: `password`
  
    addresses -> exampleTopic -> queues を展開し、ユニークなID(例: 9ae21ca4-e928-48dc-88b0-c424f37056b3)で表示されるQueue(一時キュー)が自動的に作られたことを確認します。
    このキューのAtrributesを表示して、`Temporary` が"true"となっていることを確認します。
     - `Temporary` とは、一時キューのこと。クライアントセッションが接続されてる間だけに生存するキューです。

4. 別のコマンドプロンプトを開いて、worksheet1ディレクトリから、以下のコマンドを実行して、メッセージをexampleTopicへ送信します。
    ```
    cd C:\redhat\amq7-training-master\worksheet1
    mvn verify -PtopicSender
    ```

5. topicSenderを実行すると、上記2のtopicReceiverが2件メッセージを受信して、プログラムを終了します。

6. 再度Brokerの管理コンソールへアクセスします。
   http://localhost:8161/console
    ユーザ:    `admin`
    パスワード: `password`

  addresses -> exampleTopic の配下に、先程の一時キューが消えたことを確認します。

7. Brokerインスタントを停止します。
   Brokerインスタンスの起動したコンソールにて`Ctrl + C`を押すか、別のコマンドプロンプトで以下のコマンドを実行して、Brokerインスタンスを停止してください。
    ```
    C:\redhat\broker0\bin\artemis stop
    ```

## TROUBLE-SHOOTING

### Disk Full! Blocking messageのエラー

AMQ Brokerのデータディレクトリのあるディスク容量が90パーセント以上使用されている場合、デフォルトの設定だと、Brokerがメッセージの受付をブロックします。クライアントがメッセージ送信に失敗します。

クライアント側は、以下のようなメッセージが表示されます。
> WARN: AMQ212054: Destination address=exampleQueue is blocked. If the system is configured to block make sure you consume messages on this configuration.

AMQ Brokerのログ(`broker[1,2]/log/artemis.log`)に次のようなメッセージが表示されます。
> AMQ222212: Disk Full! Blocking message production on address 'exampleQueue'. Clients will report blocked.

回避策は、`broker[1,2]\etc\broker.xml`の設定`<max-disk-usage>`を90から100に変更してください。


### KQueue is not availableの警告

Mac OS上で実行する場合、下記の警告メッセージが表示されることがあります。
> AMQ212075: KQueue is not available, please add to the classpath or configure useKQueue=false to remove this warning

この警告は、動作上の影響がありません。もし、鬱陶しいと思ったら、以下のように、`etc/broker.xml`の設定にて`useKQueue=false`を追加してください。
> <acceptor name="xxx">tcp://0.0.0.0:...?...;useKQueue=false</acceptor>
> <connector name="xxx">tcp://localhost:...?useKQueue=false</connector>


