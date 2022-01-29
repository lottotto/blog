---
title: "go言語のbyte.bufferについて"
date: 2022-01-29T17:20:36+09:00
draft: false
tags: ["Go言語"]
ShowToc: true
TocOpen: true
---

## モチベーション
Goの受け取ったデータを読んだり書いたりうまくできているのは、ReadメソッドとWriteメソッド、そしてどのメソッドを持つ、io.Reader、io.Writerといったインターフェースである。bytes.bufferをお題に色々忘れないようにメモしていく

## byteとは
1バイトの範囲（1から255）を表すデータ型。
Go言語では、ファイル処理だったり、画像だったり、リクエストだったり色々な面で`[]byte`が出てくる。
もちろん`string`も文字コードで表現されることから2バイトであるので、bytes型のスライスとして表現されることは多い。
`[]byte`と`string`の変換は下記の通り
```golang
// []bytes -> string の変換
str := string([]bytes)

// string -> []bytesの変換
b := []byte(str)

```
このようにbyte型のスライスは頻出で簡単に`string`に変換できる。このbyte型のスライスを色々簡単に操作できるようなパッケージがbytesパッケージになる。詳しくは公式ドキュメント参照。https://pkg.go.dev/byte  
bytesパッケージの中で2つだけ型が宣言されている.bytes.buffer型とreader型である。これをみていく


## bytes.bufferとは
bytes.bufferとは、readメソッドとwriteメソッドを持つ可変サイズのバイトバッファーです。
色々おさらいしていきます。bufferとは基本的には、データを一時的に記憶する場所のことを言います。なのでその一時保存領域を読んだり、一時領域に書いたりすることができるわけです。

```golang
// バッファからlen(p)サイズ文を読み取りpに格納する
package main

import (
	"bytes"
	"fmt"
)

func main() {
	str := "123456789"
	b := []byte(str)
	fmt.Println(b)
	b2 := []byte{0, 0}
	buf := bytes.NewBuffer(b)
	_, err := buf.Read(b2)
	if err != nil {
		panic(err)
	}
    // バッファからlen(p)サイズ文を読み取りpに格納する
	fmt.Println(string(b2))
}
```
実行すると12という結果が得られます。b2は2つの領域を定義しており、それが上書きされる挙動になっています。一方Writeの方は、
```golang 
package main

import (
	"bytes"
	"fmt"
)

func main() {
	str := "123456789"
	b := []byte(str)
	fmt.Println(b)
    // :;を表すバイト配列
	b2 := []byte{58, 59}
	buf := bytes.NewBuffer(b)
	_, err := buf.Write(b2)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(b2))
	// 一時領域を文字列に変換
	fmt.Println(buf.String())
}
```
上記のプログラムの実行結果としては、`123456789:;`が得られます。b2に定義していた内容が、byte.bufferの内部領域に付け加えられ、文字列変換時に標準出力に出てきます。  
こんな感じでbytes.buffer型は中にデータを可変長として持ち、入力であるバイトのスライスに対して、読み込み結果を入れるReadメソッドとスライスの内容を書き込むWriteメソッドがあり、配列のサイズなど考えずに操作可能になります。
このReadメソッドとWriteメソッドを持つということ、つまり、`io.Reader`と`io.Writer`というインターフェースで利用できるというのがGo言語で強みになっていきます。

