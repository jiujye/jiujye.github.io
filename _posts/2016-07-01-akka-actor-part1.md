---
layout: post
title:  "初探Akka Actor"
date:   2016-07-01 12:00:45 +0700
categories: [akka]
---

自從CPU從單核心發展到多核心，核心數成長飛快，由最早的雙核心發展至目前Intel提供給超級電腦使用的Xeon Phi系列61核心
或是一般伺服器版本的24核心，不過CPU每顆核心的時脈速度卻沒有顯著的成長。對於軟體設計人員而言，已經無法單純再靠CPU時脈
速度的成長來增進軟體的速度；我們該如何在這樣的環境中增進軟體的效能？這時Concurrency(並行)或是Parallelism(平行)的設計方式便為我們可以採取的一種手段。

<!--more-->
<br>
關於Concurrency的定義有多種解釋的方式，在此利用Google工程師Rob Pike(Go語言的設計者)平易近人的說明方式，
Pike認為Concurrency是同時處理(dealing with)很多事情，不同於Parallelism(平行)是同時做(doing)很多件事情。
也就是說Concurrency強調的是同時處理很多事情的結構而非同時執行這些事情。

<br>
傳統的Concurrency設計方式是以Thread為基礎，透過share state來溝通。
諸多程式語言皆是採用此種方式，像是C++、Java、C#、Python...等
。此方式為了確保正確地讀取寫入資料，因此利用了Lock機制，
然而這種設計遭遇了不少問題，例如：race condition、dead lock...等，
需要小心翼翼的設計與測試，發生的bug也有不容易重現的問題。
不少專業人士都特別針對Thread-based的Concurrency設計方式出專書介紹。

<br>
由於傳統的方式在實作上相當辛苦，大家開始從不同角度來思考，是否不要再透過share state來溝通。
像是Go語言以及Akka便是採用不同於傳統主流的Concurrent運算理論來實現。<br>
<br>
<br>
**Akka**<br>

以Akka為例，Akka為一open source工具(http://akka.io/)，
主要在幫我們更容易建置concurrent與distributed(分散)的應用程式，而Akka採用Actor Model為其核心
(Actor Model為Concurrent運算當中的一種理論)。過去提到Actor馬上直接想到的工具，便是Erlang
(Facebook曾使用Erlang打造他的聊天室系統來服務兩億位活躍使用者)。不過現在我們有了新的選擇，
Akka是利用JVM語言來實現，同時提供Java與Scala的api，而.Net的Open Source社群也實現.Net版的Akka。

<br>
在Akka Actor中Actor為基本單元，也就是說我們主要專注於設計實現符合我們需求的Actor；就同如物件導向語言，
我們專注於設計各式各樣的類別。Actor主要的特色為(以下使用原文來解釋，會更容易理解)：

* Isolated lightweight event-based processes
* Share nothing
* Communicate through async messages
* Each actor has a mailbox (message queue)
* Location transparent (distributable)
* Supervision-based failure management
<br>
**Refference:** [Björn Antonsson](http://www.slideshare.net/bantonsson/real-world-akka-actor-recipes-javaone-2013)<br>

<br>
簡單的來看Actor彼此是獨立個體，透過事件來驅動，互不分享內部資訊，只利用訊息來溝通。

<br>
在真實世界中的裡充滿了Concurrency例子，譬如像是星巴克或50嵐，前台的接單人員不斷接受客戶的訂單，
而後方多個工作人員不斷同時在處理多張不同的訂單。

<br>
以下將使用Akka Actor所提供的Java api來模擬有五個客戶向咖啡店員點單，
而接單人員將會把訂單分派給其他專門製作咖啡的店員來製作，

<br>
**設計Actor**<br>

 首先需要設計兩個Actor分別為Customer與CoffeeShopWorker，
Actor的實現需要透過繼承`UntypedActor`並override `onReceive`方法，
我們所需要完成的邏輯將寫在`onReceive`方法中，實現方式如下<br><br>


```scala
public class CoffeeShopWorker extends UntypedActor {
    @Override
    public void onReceive(Object msg) {
          …
    }
}

```

<br>
假設每個CoffeeShopWorker皆能處理訂單與製作咖啡，因此在`onReceive`方法中，
當其收到`OrderRequest`的訊息時，將會把訂單收據發送給客戶，
並轉發製作咖啡指令給其他製作咖啡的店員。然而當收到`MakeCoffee`的訊息時，
製作咖啡並把做好的咖啡交付客戶。<br><br>

```java
@Override
public void onReceive(Object msg) {
    if (msg instanceof OrderRequest ) {  //收到OrderRequest
        OrderRequest req = (OrderRequest)msg;
        sender().tell(new Receipt(),ActorRef.noSender()); //給客戶收據
            getContext().actorSelection("/user/workerRouter”) //請其他店員製作咖啡
                              .tell(new MakeCoffee(req.getRequest(),sender().path().toString()), 
					ActorRef.noSender());
        }else if(msg instanceof  MakeCoffee){ //收到MakeCoffee
            MakeCoffee  mc = (MakeCoffee)msg;
            getContext().actorSelection(mc.getCustomerPath()) //製作咖啡並交付給該客戶
                    .tell(new Coffee(mc.getCoffee()), ActorRef.noSender());
        }
    }
}

```


<br>
在Customer Actor的`onReceive`方法中，主要負責處理買咖啡，取得收據及咖啡。<br><br>

```java
@Override
public void onReceive(Object msg) {
    if (msg instanceof BuyCoffee) { //買咖啡
        BuyCoffee bc = (BuyCoffee)msg;
        //向櫃檯點單
        getContext().actorSelection("/user/Counter").tell( new OrderRequest(bc.getName()), getSelf()); 
    }else if(msg instanceof Receipt){//取得收據
        System.out.println(self().path().name() +" get "+((Receipt)msg).getInfo());
    }else if(msg instanceof Coffee){//取得咖啡
        System.out.println( self().path().name() +" get "+((Coffee)msg).getName());
}

```

<br>
此外我們可以覆寫Customer Actor的`preStart`方法在其中設定timer讓Customer Actor會自動買咖啡。<br><br>

```java
@Override
public void preStart() {
    getContext().system().scheduler().scheduleOnce(
            Duration.create(500, TimeUnit.MILLISECONDS),
            getSelf(), new BuyCoffee(this.coffee), getContext().dispatcher(), null);
}

```

<br>
**建立Actor**<br>

主要的邏輯設計完成後，我們該如何建立Actor讓其運作。
首先我們需要建立Actor System。<br><br>

```java
ActorSystem system = ActorSystem.create("CoffeeShop”);
```

<br>
接著利用system的`actorOf`來建立Actor，當中需要的參數為Actor的設定物件`Props`，以及Actor名稱。
在建立Actor時，我們可以獲得`ActorRef`，這是Actor的參考，因為我們不能直接操作Actor實體，
所有的操作都只能透過`ActorRef`，Actor實體是各自獨立受到保護的。<br><br>

```java
ActorRef counter = system.actorOf(CoffeeShopWorker.props(),"Counter”);  //建立前台接單人員
```

<br>
此外在這個例子中我們還利用了`Router`的功能，建立三個咖啡製作人員，
透過`Router`使用`RoundRobin`的方式來分派製作咖啡指令。<br><br>

```java
system.actorOf(new RoundRobinPool(3).props(CoffeeShopWorker.props()), "workerRouter");
```

<br>
**傳送訊息**<br>

Actor彼此之間是透過訊息來溝通，所以必須使用`ActorRef`來傳遞。`ActorRef`除來在建立時可以取得，
也可透過`actorSelection`方法來取得，當中所需的參數為Actor的path。<br><br>

```java
getContext().actorSelection("/user/Counter").tell( new OrderRequest(bc.getName()), getSelf());
```

<br>
**執行方式**<br>

在main方法中建立一個前台接單人員Actor，三個咖啡製作人員Actor以及五個客戶Actor。
客戶Actor在啟動前會設定timer自動傳遞買咖啡的指令，向台接單人員購買咖啡。<br><br>

```java
public static void main(final String[] args){

    ActorSystem system = ActorSystem.create("CoffeeShop");

    system.actorOf(CoffeeShopWorker.props(),"Counter");
    system.actorOf(new RoundRobinPool(3).props(CoffeeShopWorker.props()), "workerRouter");

    system.actorOf(Customer.props("Latte"),"Customer1");
    system.actorOf(Customer.props("Black Coffee"),"Customer2");
    system.actorOf(Customer.props("Cappuccino"),"Customer3");
    system.actorOf(Customer.props("Iced Coffee"),"Customer4");
    system.actorOf(Customer.props("Iced Tea"),"Customer5");

}
```

<br>
**執行結果**<br>

利用輸出可以看出，客戶Actor在啟動前透過設定好的timer自動傳遞買咖啡的指令，向前台接單人員購買咖啡，
而三個咖啡製作人員則同時處裡不同客戶的訂單。 不過由於Akka內部Dispatcher配置thread機制的關係，
我們每次執行的輸出結果的順序會有所不同。<br><br>

```
2016-07-01 10:16:29.133 Counter handle Customer2 order.
2016-07-01 10:16:29.134 Customer2 get Receipt  
2016-07-01 10:16:29.135 Counter handle Customer5 order.
2016-07-01 10:16:29.135 Counter handle Customer4 order.
2016-07-01 10:16:29.136 Worker $b make Iced Tea.
2016-07-01 10:16:29.136 Customer4 get Receipt  
2016-07-01 10:16:29.136 Worker $c make Iced Coffee.
2016-07-01 10:16:29.136 Counter handle Customer1 order.
2016-07-01 10:16:29.137 Customer1 get Receipt  
2016-07-01 10:16:29.137 Customer5 get Receipt  
2016-07-01 10:16:29.137 Customer5 get Iced Tea
2016-07-01 10:16:29.137 Customer4 get Iced Coffee
2016-07-01 10:16:29.138 Worker $a make Black Coffee.
2016-07-01 10:16:29.138 Customer2 get Black Coffee
2016-07-01 10:16:29.138 Worker $a make Latte.
2016-07-01 10:16:29.138 Counter handle Customer3 order.
2016-07-01 10:16:29.138 Customer1 get Latte
2016-07-01 10:16:29.138 Worker $b make Cappuccino.
2016-07-01 10:16:29.138 Customer3 get Receipt  
2016-07-01 10:16:29.138 Customer3 get Cappuccino
```

<br>
**結語**<br>

藉由這個簡單例子的介紹，我們可以了解Akka Actor主要的特色，與Concurrent Actor Model運作的基本原理。
但這只是個開頭，許多Akka重要的概念與細節都沒有特別介紹，主要是希望藉由這個開頭讓大家對於Akka產生興趣，
未來將會有更多關於Akka的介紹。Akka Actor詳細資訊可參考[http://doc.akka.io/docs/akka/current/java/untyped-actors.html](http://doc.akka.io/docs/akka/current/java/untyped-actors.html)
。上述完整程式碼請見[https://github.com/jiujye/examples/tree/master/akka-actor-example1](https://github.com/jiujye/examples/tree/master/akka-actor-example1)。


