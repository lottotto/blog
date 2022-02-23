---
title: "Goのポインタについて"
date: 2022-02-23T15:36:55+09:00
draft: true
tags: [""]
ShowToc: true
TocOpen: true
---

## Go言語のポインタについて

### アドレスとは
アドレスとは、変数の内容が保存されるメモリの場所を示す16進数の値である。変数名の前に`&`をつけることでアドレスを提供する.このように定義された変数をポインタ変数という

```go
a := "aaaa"
b := &a // bはポインタ変数になる
```

### ポインタとは
仕組みとしてはアドレスが格納されているポインタ変数に対して、*をつけることで、アドレスの中身にアクセスすることができる。ポインタとは、格納されている値の型情報とアドレスから成り立っている.
```go
a := "aaaa"
b := &a // bはポインタ変数になる
fmt.Println(*b)　 // これでaaaaと出力される。
```

### ポインタのメリット
ポインタを利用することで参照渡しが実現できる。参照渡しとは、アドレスを渡すことで変数自体を渡すことである。参照私に対する言葉として値渡しがあるが、値渡しとは、変数の中身をコピーして関数にわたしていることになる。
C言語などと異なり、ポインタに対しての加算はできないので気をつけること。
```go
package main

import "fmt"

func incrementPassByValue(i int) {
	i = i + 1
	fmt.Println(i)
}
func incrementPassByReference(i *int) {
	i = i + 1 //ポインタの加算はできないので注意すること
	fmt.Println(i)
}

func main() {
	i := 100
	fmt.Println(i)               // 100
	incrementPassByValue(i)      // 101
	fmt.Println(i)               // 100
	incrementPassByReference(&i) // 101、、と行きたいところだがコンパイルできない。
	fmt.Println(i)               // 101

}

```

### ポインタレシーバとは
ではポインタとして変数を変えたいときはどうするべきか。これはポインタレシーバを利用するのが良いと思う
```
package main

import "fmt"

type Person struct {
	Age  int
	Name string
}

func (p Person) increment() {
	p.Age = p.Age + 1
}

func (p *Person) incrementByReference() {
	p.Age = p.Age + 1
}

func main() {
	person := &Person{Age: 24, Name: "田中太郎"}
	person.increment()
	fmt.Println(person.Age) // 24のまま
	person.incrementByReference()
	fmt.Println(person.Age) // 25になる
}
```
### まとめ
中のメソッドを更新するときはPointerレシーバーを使おう。