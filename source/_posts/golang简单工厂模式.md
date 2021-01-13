---
title : Golang-简单工厂模式
date: 2021-01-13 19:39:12
categories: Golang
tags :
  - golang
  - gin
  - go
---
# 简单工厂模式
简单来说，简单工厂模式有一个具体的工厂类，可以生成多个不同的产品，属于创建型设计模式。简单工厂模式不在 GoF 23 种设计模式之列。

## 优点和缺点
### 优点：
1.工厂类包含必要的逻辑判断，可以决定在什么时候创建哪一个产品的实例。客户端可以免除直接创建产品对象的职责，很方便的创建出相应的产品。工厂和产品的职责区分明确。  
2.客户端无需知道所创建具体产品的类名，只需知道参数即可。  
3.可以引入配置文件，在不修改客户端代码的情况下更换和添加新的具体产品类。
### 缺点：
1.简单工厂模式的工厂类单一，负责所有产品的创建，职责过重，一旦异常，整个系统将受影响。且工厂类代码会非常臃肿，违背高聚合原则。  
使用简单工厂模式会增加系统中类的个数（引入新的工厂类），增加系统的复杂度和理解难度  
2.系统扩展困难，一旦增加新产品不得不修改工厂逻辑，在产品类型较多时，可能造成逻辑过于复杂  
3.简单工厂模式使用了 static 工厂方法，造成工厂角色无法形成基于继承的等级结构。
### 应用场景
对于产品种类相对较少的情况，考虑使用简单工厂模式。使用简单工厂模式的客户端只需要传入工厂类的参数，不需要关心如何创建对象的逻辑，可以很方便地创建所需产品。

## demo
``` go
package main

import "fmt"

func main() {
	// A car has four wheels
	NewTheTrafficTools("c").ToGetParts()
	// A bicycle has two wheels
	NewTheTrafficTools("b").ToGetParts()
}

type TheTrafficTools interface {
	ToGetParts()
}

type Car struct {
}

func (c *Car) ToGetParts() {
	fmt.Println("A car has four wheels")
}

type Bicycle struct {
}

func (b *Bicycle) ToGetParts() {
	fmt.Println("A bicycle has two wheels")
}

func NewTheTrafficTools(name string) TheTrafficTools {
	switch name {
	case "c":
		return &Car{}
	case "b":
		return &Bicycle{}
	default:
		return nil
	}
}
```
