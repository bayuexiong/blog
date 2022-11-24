---
title: "手写一个Go Json库 - 解析篇"
date: "2022-10-10"
---

大概是2018年的时候看到 Milo Yip 大神的[从零开始的 JSON 库教程](https://github.com/miloyip/json-tutorial)，便开始着手跟着教程开始学习，不过可能拖延症爆发，这件事就一直延误下来了。

我写的解析器十分简单，读取第一个非空白的字符->判断类型->验证类型是否正确->设置值->判断结束，是一个递归下降解析器（recursive descent parser）

首先解析器的定义接口

```go
// 定义解析API
type parse interface {
	// 解析入库
	parser() (*jsonValue, error)
	// 解析value，可以递归调用
	parserValue() (*jsonValue, error)
	// 解析 null、true、false
	parseLiteral(literal []byte, v *jsonValue, valueType ValueType) error
	// 解析数字
	parseNumber(v *jsonValue) error
	// 解析字符串
	parseString(v *jsonValue) error
	// 解析数组
	parseArray(v *jsonValue) error
	// 解析对象
	parseObject(v *jsonValue) error
}
```

中间值的实现

```go
// jsonValue 解析json的中转值，[]byte > jsonValue -> var
type jsonValue struct {
	object
	array
	s         []byte
	n         float64
	valueType ValueType
}

type object struct {
	size   int
	keys   []*jsonValue
	values []*jsonValue
}

type array struct {
	len    int
	values []*jsonValue
}
```

Go 版本实现有个缺陷，Lept Json 是用一个公用的栈来存储string，array，object的解析，而我偷懒每种类型解析用了不同的空间存储中间值，这个后期需要优化（也可能不需要优化

详细代码包括单元测试

[Lept Json 的 GO 版本]([https://github.com/bayuexiong/go-json](https://github.com/bayuexiong/go-json))

