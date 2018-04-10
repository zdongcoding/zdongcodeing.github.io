---
title: Kotlin基础语法
copyright: true
categories: android
date: 2018-01-03 11:35:39
tags: Kotlin
---

1.定义包名（简单和Java 基本一样）
2.定义函数
> fun 开头表示函数，函数的返回值和形参（java的名称）类型 都是名字后面 ：TYPE (类型都是名称后面加：) 语句结尾分号可以忽略不写
```java
fun sum(a: Int , b: Int) : Int{
	return a + b
}

fun main(args: Array<String>) {
  print("sum of 3 and 5 is ")
  println(sum(3, 5))
}

fun sum(a: Int, b: Int) = a + b     // 该函数只有一个表达式函数体以及一个自推导型的返回值：
```
当函数的不需要返回值时（返回无意义的值时）可以忽略不写  （：Unit）

更多详细讲解

3.定义局部变量
声明常量（val）
```java
fun main(args: Array<String>) {
  val a: Int = 1  // 立即初始化
  val b = 2   // 推导出Int型
  val c: Int  // 当没有初始化值时必须声明类型
  c = 3       // 赋值
  println("a = $a, b = $b, c = $c")   //使用了字符串模板
}
```
声明变量
```java
fun main(args: Array<String>) {
  var x = 5 // 推导出Int类型
  x += 1
  println("x = $x")
}
```
4.注释 （java 一样）
5.字符串模板
> 可以使用变量名，表达式，
```
fun main(args: Array<String>) {
  var a = 1
  // 使用变量名作为模板:
  val s1 = "a is $a"

  a = 2
  // 使用表达式作为模板:
  val s2 = "${s1.replace("is", "was")}, but now is $a"
  println(s2)
}
```
6.条件表达式
> if  else   when(取代了switch)
if else 这就不多说了， 简单

when 比 switch功能多，  可以使用 常量 和使用表达式 ，in, !in (检查值是否值在一个集合), is !is(判断值是否是某个类型) ,可以用来代替 if-else if 
```java
when (x) {
  1 -> print("x == 1")
  2 -> print("x == 2")
  else -> { //Note the block
		print("x is neither 1 nor 2")
	}
}
```
for 循环
```java
for (item in collection)
	print(item)

for (item: Int in ints){
	//...
}
//类似java中的for(int i:array) 但是kotlin更强大    
//java中的切点无法获取 index
//Kotlin可以获取 "indices"
for (i in array.indices)
	print(array[i])
```
while,do while break,continue  和 java 一样

7.使用可空变量以及空值检查

下面的函数是当 str 中不包含整数时返回空:

```java
fun parseInt(str : String): Int?{
	//...
}
```
使用一个返回可空值的函数：

```java
fun parseInt(str: String): Int? {
  return str.toIntOrNull()
}

fun printProduct(arg1: String, arg2: String) {
  val x = parseInt(arg1)
  val y = parseInt(arg2)

  // 直接使用 x*y 会产生错误因为它们中有可能会有空值
  if (x != null && y != null) {
    // x 和 y 将会在空值检测后自动转换为非空值
    println(x * y)
  }
  else {
    println("either '$arg1' or '$arg2' is not a number")
  }    
}

fun main(args: Array<String>) {
  printProduct("6", "7")
  printProduct("a", "7")
  printProduct("a", "b")
}
```
或者这样
```java
fun parseInt(str: String): Int? {
  return str.toIntOrNull()
}

fun printProduct(arg1: String, arg2: String) {
  val x = parseInt(arg1)
  val y = parseInt(arg2)

  // ...
  if (x == null) {
    println("Wrong number format in arg1: '${arg1}'")
    return
  }
  if (y == null) {
    println("Wrong number format in arg2: '${arg2}'")
    return
  }

  // x 和 y 将会在空值检测后自动转换为非空值
  println(x * y)
}
```
默认情况下都是空值检查 即 没有 ？ 当为空时不执行该代码片段

8.for 循环
```java
fun main(args: Array<String>) {
  val items = listOf("apple", "banana", "kiwi")
  for (item in items) {
    println(item)
  }
}
fun main(args: Array<String>) {
  val items = listOf("apple", "banana", "kiwi")
  for (index in items.indices) {   //indices 是角标
    println("item at $index is ${items[index]}")
  }
}
```
9.ranges
>检查 in 操作符检查数值是否在某个范围内：
```java
fun main(args: Array<String>) {
  val x = 10
  val y = 9
  if (x in 1..y+1) {
      println("fits in range")
  }
}
```
检查数值是否在范围外
```java
fun main(args: Array<String>) {
  val list = listOf("a", "b", "c")

  if (-1 !in 0..list.lastIndex) {
    println("-1 is out of range")
  }
  if (list.size !in list.indices) {
    println("list size is out of valid list indices range too")
  }
}
```
在范围内迭代
```java
fun main(args: Array<String>) {
  for (x in 1..5) {
    print(x)
  }
}
```
或者使用步进：
```java
fun main(args: Array<String>) {
  for (x in 1..10 step 2) {
    print(x)
  }
  for (x in 9 downTo 0 step 3) {
    print(x)
  }
}
```