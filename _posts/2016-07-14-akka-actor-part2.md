---
layout: post
title:  Actor運作原理
date:   2016-07-14 10:00:45 +0800
categories: [akka]
---

透過Akka創作者Jonas Bonér於研討會中對於Actor的講解，
讓我們可以快速理解Actor的運作方式。以下為的Bonér的演講內容
(5分55秒到9分12秒為解釋Actor的部分)。

<!--more-->
<br>

<iframe width="500" height="315" src="https://www.youtube.com/embed/UY3fuHebRMI?start=355" frameborder="0" allowfullscreen></iframe>

<br>
在這將近4分鐘的內容中，Bonér利用投影片清楚而簡單的說明了Actor的基本原理。

<br>
經由影片中的投影片，我們可以了解每個Actor有其本身的狀態與行為，
如同物件導向的物件一樣，不過每個Actor都有其各自的Mailbox。
Actor彼此不分享其內部的狀態與行為，只透過訊息來溝通，
而訊息傳遞至各自的Mailbox中，Actor一次只處理一個訊息。

<br>
Actor可以是thread-based，不過大部分的Actor System是採用event-based的機制，
也就是說Actor是運行在event-based的thread上。Actor System透過內部的排程器將Actor從pool中取出，
運行在thread pool中的某個thread上，當其時間到了，
Actor System會將其放回pool中，而其他Actor可以繼續運行在該thread。

<br>
以上為Bonér對於Actor運作方式的一般性說明。
至於在Akka中專門負責讓Actor運作的角色為`MessageDispatcher`，
Akka有三種類型的MessageDispatcher，分別為`Dispatcher`，`PinnedDispatcher`與`BalancingDispatcher`，以下為簡單的說明。
<br>

* Dispatcher：預設的MessageDispatcher，運作的方式如同上述，是採用event-based的機制，
背後採用了Java的ExecutorService。

* PinnedDispatcher：當你不想讓Actor共用thread時，採用PinnedDispatcher會配置給每個Actor獨一無二的thread。

* BalancingDispatcher：在BalancingDispatcher下，所有的Actor會共用一個Mailbox，此Dispatcher將會確保訊息交由閒置的Actor來處理。

<br>
**詳細的資料可參考** [Akka Dispatcher](http://doc.akka.io/docs/akka/2.4/scala/dispatchers.html)

