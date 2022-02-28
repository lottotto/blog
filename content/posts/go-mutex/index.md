---
title: "Go言語の排他制御について"
date: 2022-02-23T16:11:43+09:00
draft: false
tags: ["Go"]
ShowToc: true
TocOpen: true
---

## Go言語の排他制御について
排他制御とは、マルチスレッドで同じ変数を参照しているとき、どっちも書き込もうとしたら変更が衝突して意図していない結果が得られることである。
そのためには、変数をロックしておくことが必要になる。実現方法としては、`sync`パッケージの中の`Mutex`を利用することである。
https://pkg.go.dev/sync

## mutex
mutexは相互排他ロックのことをいいます。mutexの0値は、ロック解除されたことを示します。 mutexは、最初の使用後にコピーしてはなりません。
```go 
type Mutex struct {
	state int32
	sema  uint32
}
```
この構造体に対して2つのメソッドが定義されています。`Lock()`と`Unlock()`です。それぞれ、一つしかない変数など排他制御の仕組みを利用してロック状態、非ロック状態を定義しています。
実際の使用例はTour of Go のWebCraelerになります。

↓コード全体
```go
package main

import (
	"fmt"
	"sync"
)

type Fetcher interface {
	Fetch(url string) (body string, urls []string, err error)
}

type Result struct {
	// 追加。なぞったかどうかを覚えておくスライスを定義
	// 排他制御の情報を持っておく、mu変数を定義
	resultMap map[string]bool
	mu        sync.Mutex
}

func (rm *Result) isCrawled(url string) bool {
	// sync.Mutexを更新するので、ポインタレシーバーを利用する
	// すでに閲覧したかどうかを確認する。まずはロックをする
	rm.mu.Lock()
	defer rm.mu.Unlock()
	if _, ok := rm.resultMap[url]; ok {
		return true
	} else {
		rm.resultMap[url] = true
		return false
	}
}

// すべてのGoRoutineが完了するまでまつ
var wg sync.WaitGroup

func Crawl(url string, depth int, fetcher Fetcher, resultMap *ResultMap) {
	defer wg.Done() //Crawlの処理が終わったら、WaitGroupの待機数を減らす
	if depth <= 0 {
		return
	}
	if resultMap.isCrawled(url) {
		return
	} else {
		body, urls, err := fetcher.Fetch(url)
		if err != nil {
			fmt.Println(err)
			return
		}
		fmt.Printf("found: %s %q\n", url, body)
		for _, u := range urls {
			wg.Add(1) // goRoutineに流す前にまずはWaitGroupを増やす
			go func(u string) {
				Crawl(u, depth-1, fetcher, resultMap) // L38あたりのwg.Doneでwgは減らされる
			}(u)
		}
	}
	return
}

func main() {
	// resultMap := &ResultMap{}
	resultMap := &Result{resultMap: make(map[string]bool)}
	wg.Add(1) // 最初のgoroutineに流すので、まずは1つ増やす
	go Crawl("https://golang.org/", 4, fetcher, resultMap)
	wg.Wait() // すべてのWaitGroupがなくなるまでまつ
}

// // fakeFetcher is Fetcher that returns canned results.
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
	if res, ok := f[url]; ok {
		return res.body, res.urls, nil
	}
	return "", nil, fmt.Errorf("not found: %s", url)
}

var fetcher = fakeFetcher{
	"https://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"https://golang.org/pkg/",
			"https://golang.org/cmd/",
		},
	},
	"https://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"https://golang.org/",
			"https://golang.org/cmd/",
			"https://golang.org/pkg/fmt/",
			"https://golang.org/pkg/os/",
		},
	},
	"https://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
	"https://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
}

```

```go 
func (rm *Result) isCrawled(url string) bool {
	// sync.Mutexを更新するので、ポインタレシーバーを利用する
	// すでに閲覧したかどうかを確認する。まずはロックをする
	rm.mu.Lock()
	defer rm.mu.Unlock()
	if _, ok := rm.resultMap[url]; ok {
		return true
	} else {
		rm.resultMap[url] = true
		return false
	}
}
```
mutexは1つのGoroutineがアクセスできるような情報を持つらしく、これによって、resultにもアクセスできなくなるっぽい。（つまり読み取れなくなる。）
もし、resultMapの中にurlがあればすでに巡回済身として、trueを戻し、そうじゃなかったら、巡回リストに書き込み、falseを戻す。

``` go
var wg sync.WaitGroup

func Crawl(url string, depth int, fetcher Fetcher, resultMap *ResultMap) {
	defer wg.Done() //Crawlの処理が終わったら、WaitGroupの待機数を減らす
	if depth <= 0 {
		return
	}
	if resultMap.isCrawled(url) {
		return
	} else {
		body, urls, err := fetcher.Fetch(url)
		if err != nil {
			fmt.Println(err)
			return
		}
		fmt.Printf("found: %s %q\n", url, body)
		for _, u := range urls {
			wg.Add(1) // goRoutineに流す前にまずはWaitGroupを増やす
			go func(u string) {
				Crawl(u, depth-1, fetcher, resultMap) // L38あたりのwg.Doneでwgは減らされる
			}(u)
		}
	}
	return
}

func main() {
	// resultMap := &ResultMap{}
	resultMap := &Result{resultMap: make(map[string]bool)}
	wg.Add(1) // 最初のgoroutineに流すので、まずは1つ増やす
	go Crawl("https://golang.org/", 4, fetcher, resultMap)
	wg.Wait() // すべてのWaitGroupがなくなるまでまつ
}
```
Crawl関数については、関数終了時にWaitGroupを減らす処理を入れている。そもそもWaitGroupとは、すべてのGoroutineが終了するまで待つ、といった制御が可能になるものである。
wg.Addやwg.Doneを入れる場所を間違えるとうまくいかないので注意すること。（特に再帰して関数を呼び出す場合）

### まとめ
- Goroutineを利用したマルチスレッドプログラムで共通の変数を更新にかかるときは、syncパッケージによるmutexを利用する。Lockメソッドを使うことで、(おそらく同階層にある変数に対して)そのロックしたGoRoutineしかアクセスできなくなり、UnlockされるまでアクセスしようとするGoroutineは待つことになる。
- 多数のGoroutineを利用する際は、WaitGroupを利用してすべてのGoroutineの処理を待つことができる。wg.Doneやwg.addを入れるタイミングはとても重要。