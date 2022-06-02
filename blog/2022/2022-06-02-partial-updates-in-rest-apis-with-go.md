categories:
- 技术

tags:
- Go

title: 如何用 Go 实现 REST API 部分更新
---

## 一、问题

假设我们有一条数据如下：

```
GET /people/1 HTTP/1.1

{
    "name": "John",    
    "age": 20,
    "address": {
        "country": "China",
        "province": "Guangdong",
        "city": "Shenzhen",
    }
}
```

现在我们想更新 John 的年龄和所在城市，于是发起了一个请求：

```
PATCH /people/1 HTTP/1.1

{
    "age": 25,
    "address": {
        "city": "Guangzhou",
    }
}
```

作为 Go 服务端开发人员，我们如何才能正确处理这个部分更新请求呢？


## 二、JSON 与 Go 的零值

乍一看并不难，我们立马写下了结构体定义：

```go
type Address struct {
	Country  string `json:"country"`
	Province string `json:"province"`
	City     string `json:"city"`
}

type Person struct {
	Name    string  `json:"name"`
	Age     int     `json:"age"`
	Address Address `json:"address"`
}
```

JSON 反序列化？自然也不在话下：

```go
blob := []byte(`{"age": 25, "address": {"city": "Guangzhou"}}`)
var person Person
_ = json.Unmarshal(blob, &person)
fmt.Printf("person: %+v\n", person)
```

对应的输出结果（[Go Playground](https://go.dev/play/p/8Kod8cgvwXF)）：

```
person: {Name: Age:25 Address:{Country: Province: City:Guangzhou}}
```

很显然，如果我们直接用 person 去更新 John 的数据，他的姓名、所在国家和省份都会被清空！

那服务端该如何正确识别客户端的原始意图呢？具体到 John 的例子，在 Go 中如何做到 “只更新他的年龄和所在城市” 呢？


## 三、业界通用解法

据我所知，对于上述问题，业界通常有以下三种解法。

### 使用指针

因为 Go 的[零值特性][1]，普通类型无法表达 “未初始化” 的状态，典型解法就是使用指针。

采用指针后，上面的结构体定义将变成：

```go
type Address struct {
	Country  *string `json:"country"`
	Province *string `json:"province"`
	City     *string `json:"city"`
}

type Person struct {
	Name    *string  `json:"name"`
	Age     *int     `json:"age"`
	Address *Address `json:"address"`
}
```

再次进行 JSON 反序列化：

```go
blob := []byte(`{"age": 25, "address": {"city": "Guangzhou"}}`)
var person Person
_ = json.Unmarshal(blob, &person)
fmt.Printf("person: %+v, address: %+v\n", person, person.Address)
```

对应的输出结果（[Go Playground](https://go.dev/play/p/7Wm-rrymwdA)）：

```
person: {Name:<nil> Age:0xc000018218 Address:0xc00000c138}, address: &{Country:<nil> Province:<nil> City:0xc0000103f0}
```

可以看到只有 Age 和 Address.City 的值不为 nil，于是我们只需要更新不为 nil 的字段即可：

```go
func (p *Person) Update(other Person) {
	if other.Name != nil {
		p.Name = other.Name
	}

	if other.Age != nil {
		p.Age = other.Age
	}

	if addr := other.Address; addr != nil {
		// 使用指针的副作用
		if p.Address == nil {
			p.Address = new(Address)
		}

		switch {
		case addr.Country != nil:
			p.Address.Country = addr.Country
		case addr.Province != nil:
			p.Address.Province = addr.Province
		case addr.City != nil:
			p.Address.City = addr.City
		}
	}
}
```

参考完整代码（[Go Playground](https://go.dev/play/p/7Hqu-B5ZZDd)）不难发现，使用指针后的 Person 结构体，操作起来会非常繁琐（尤其是 “初始化” 操作）。

### 客户端维护的 FieldMask

受 Protocol Buffers 设计的影响，另一种较为流行的做法是在请求中新增一个 [FieldMask][2] 参数，用来补充说明需要更新的字段名称。

```go
type Address struct {
	Country  string `json:"country"`
	Province string `json:"province"`
	City     string `json:"city"`
}

type Person struct {
	Name    string  `json:"name"`
	Age     int     `json:"age"`
	Address Address `json:"address"`
}

type UpdatePersonRequest struct {
	Person    Person `json:"person"`
	FieldMask string `json:"field_mask"`
}
```

```go
blob := []byte(`{"person": {"age": 25, "address": {"city": "Guangzhou"}}, "field_mask": "age,address.city"}`)
var req UpdatePersonRequest
_ = json.Unmarshal(blob, &req)
fmt.Printf("req: %+v\n", req)
```

对应的输出结果（[Go Playground](https://go.dev/play/p/Wu27HbQS-KI)）：

```
req: {Person:{Name: Age:25 Address:{Country: Province: City:Guangzhou}} FieldMask:age,address.city}
```

有了 FieldMask 的补充说明，服务端就能正确进行部分更新了。当然对于客户端而言，FieldMask 其实是多余的，而且维护成本也不低（特别是待更新字段较多时），这也是我认为该方案最明显的一个不足之处。

### 改用 JSON Patch

前面讨论的方案，本质上都是 [JSON Merge Patch][3] 风格的。部分更新还有另外一个比较有名的风格，那就是 [JSON Patch][4]。

具体到 John 的例子，部分更新请求变成了：

```
PATCH /people/1 HTTP/1.1

[
    { 
        "op": "replace", 
        "path": "/age", 
        "value": 25
    },
    {
        "op": "replace",
        "path": "/address/city",
        "value": "Guangzhou"
    }
]
```

相比于前面的解法而言，该解法的主要缺点是 PATCH 请求体跟待更新文档的 JSON 数据格式差异太大，表达上不太符合直觉。


## 四、服务端维护的 FieldMask

如果我们坚持 JSON Merge Patch 风格的部分更新，综合来看「客户端维护的 FieldMask」是相对较好的方案。那有没有可能进一步规避该方案的不足，即不增加客户端的维护成本呢？经过一段时间的研究和思考，我认为答案是肯定的。

有经验的读者可能会发现，Go 的 JSON 反序列化其实有两种：

- 将 JSON 反序列化为结构体（优势：操作直观方便；不足：有零值问题）
- 将 JSON 反序列化为 `map[string]interface{}`（优势：能够准确表达 JSON 中有无特定字段；不足：操作不够直观方便）

可想而知，如果我们直接把 Person 从结构体改为 `map[string]interface{}`，操作体验可能会比使用带指针的结构体更糟糕！

那如果我们只是把 map 作为一个反序列化的中间结果呢？比如：

1. 首先将 JSON 反序列化为 map
2. 然后用 map 来充当（服务端维护的）FieldMask
3. 最后将 map 解析为结构体（幸运的是，已经有现成的库 [mapstructure](https://github.com/mitchellh/mapstructure) 可以做到！）

通过一些探索和试验，结果表明上述想法是可行的。为此，我还专门开发了一个小巧的库 [fieldmask][5]，用来辅助实现基于该想法的部分更新。

具体到 John 的例子，借助 [fieldmask][5] 库，结构体可以定义成最自然的方式（不需要使用指针）：

```go
type Address struct {
	Country  string `json:"country"`
	Province string `json:"province"`
	City     string `json:"city"`
}

type Person struct {
	Name    string  `json:"name"`
	Age     int     `json:"age"`
	Address Address `json:"address"`
}

type UpdatePersonRequest struct {
	Person
	FieldMask fieldmask.FieldMask `json:"-"`
}

func (req *UpdatePersonRequest) UnmarshalJSON(b []byte) error {
	if err := json.Unmarshal(b, &req.FieldMask); err != nil {
		return err
	}
	return mapstructure.Decode(req.FieldMask, &req.Person)
}
```

注意，其中最核心的代码是 `UnmarshalJSON`。对应的更新逻辑如下：

```go
person := Person{
	Name: "John",
	Age:  20,
	Address: Address{
		Country:  "China",
		Province: "Guangdong",
		City:     "Shenzhen",
	},
}

blob := []byte(`{"age": 25, "address": {"city": "Guangzhou"}}`)
req := new(UpdatePersonRequest)
if err := json.Unmarshal(blob, req); err != nil {
	fmt.Printf("err: %#v\n", err)
}

// Update top-level fields.
if req.FieldMask.Has("name") {
	person.Name = req.Name
}

if req.FieldMask.Has("age") {
	person.Age = req.Age
}

// Update address subfields.
addressFM, ok := req.FieldMask.FieldMask("address")
if !ok {
	return
}

switch {
case addressFM.Has("country"):
	person.Address.Country = req.Address.Country
case addressFM.Has("province"):
	person.Address.Province = req.Address.Province
case addressFM.Has("city"):
	person.Address.City = req.Address.City
default:
	// Empty address.
	person.Address = req.Address
}
```

个人觉得，相比其他方案而言，上述代码实现非常简单、自然（如果还有优化空间，欢迎指正和讨论👏🏻）。

当然该方案也不是完美的，目前来说，我认为至少有一个瑕疵就是需要两次解码：JSON -> map -> 结构体。


## 五、相关阅读

- [Go, REST APIs, and Pointers][6]
- [Practical API Design at Netflix, Part 2: Protobuf FieldMask for Mutation Operations][7]
- [REST: Partial updates with PATCH][8]


[1]: https://dave.cheney.net/2013/01/19/what-is-the-zero-value-and-why-is-it-useful
[2]: https://google.aip.dev/134
[3]: https://datatracker.ietf.org/doc/html/rfc7396
[4]: https://datatracker.ietf.org/doc/html/rfc6902
[5]: https://github.com/RussellLuo/fieldmask
[6]: https://willnorris.com/2014/05/go-rest-apis-and-pointers/
[7]: https://netflixtechblog.com/practical-api-design-at-netflix-part-2-protobuf-fieldmask-for-mutation-operations-2e75e1d230e4
[8]: https://www.mscharhag.com/api-design/rest-partial-updates-patch
