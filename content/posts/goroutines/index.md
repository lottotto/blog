---
title: "今更ながらGo Routineを勉強する"
date: 2022-02-19T19:47:24+09:00
draft: false
tags: [""]
ShowToc: true
TocOpen: true
---

- Goroutine
Goのランタイムに管理される軽量なスレッドになる。スレッド数やメモリ管理などは、Goがよしなにやってくれるのでそこまで気にしなくても大丈夫（らしい）  
go func(x,y)という形で宣言することで、別スレッドで関数を実施できる。mainのGoroutineから外れて、別スレッドに渡される。サンプルは下記の通りになる。

```go
package main

import (
	"fmt"
	"time"
)

func main() {

	go func() { // 別goroutine に渡されて、実行される
		fmt.Println("Hello Goroutine")
	}()

	fmt.Println("Hello Main")
	time.Sleep(1 * time.Second)
}
```
実行結果。別のGorutineで実行されているので、Hello Goroutineが先に帰ってくるかもしれない
```bash
% go run main.go
Hello Main
Hello Goroutine
```

- Channel
チャネルオペレータの <- を用いて値の送受信ができる通り道です。この通り道というのはGoroutineとGoroutineを繋ぐという意味での通り道です。下記のように値を送ったり、受信したりすることができる。channelの値格納ポリシーは先入先出であるので、キューとして利用することができる。
```go
ch <- v // channelへvを送信
v := <- ch // channelから受信したものをvという変数にする
```

channelの定義は下記の通りにできる。２番面の引数は channelをバッファとして利用する際に必要で、格納できる値の個数となる。バッファサイズのデフォルトは0となっており、goroutineでchannelに値を送信後、受信するまでデットロック状態になります。
```
ch := make(chan int, 100)
```
channelは開いているか閉じているか、の状態を持っています。これによりfor文などでの対応が可能

```
v, ok :=  <- ch
```

```go
package main

import (
	"fmt"
)

func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	close(c)
}

func main() {
	c := make(chan int, 10)
	go fibonacci(cap(c), c) // cap関数でchannelのサイズを取得できる
	for i := range c {  // channelがクローズになるまで待ち続ける
		fmt.Println(i)
	}
}
```

基本的な使い方としてはこれで大丈夫だと思う。適宜mutexを利用した排他制御なども見てください

