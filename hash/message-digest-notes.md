# 常用信息摘要算法的注意事项

## 安全强度

| 函数名称 | 安全强度(位) | 散列值长度(位) |
|:-------:|:-------:|:-------:|
| MD5 | <=18 | 128 |
| SHA1 | <58 | 160 |
| SHA224 | 112 | 224 |
| SHA256 | 128 | 256 |
| SHA512/224 | 112 | 224 |
| SHA512/256 | 128 | 256 |
| SHA384 | 192 | 384 |
| SHA512 | 256 | 512 |

安全强度是信息摘要算法衡量安全性的指标，通常用`位`来表述，代表的意义就是**N 位的安全强度表示破解一个算法需要 2^N(2 的 N 次方) 次的运算**

根据`NIST`建议，`112`位安全强度的信息摘要算法可以在`2030`年之前使用，对照表中记录`MD5` `SHA1`已经不适用于安全应用，`SHA224` `SHA512/224`在`2030`年之前相对安全，其他的可以沿用至`2030`年之后

## 处理长度

| 函数名称 | 处理长度(位) |
|:-------:|:-------:|
| MD5 | 2^64 |
| SHA1 | 2^64 |
| SHA224 | 2^64 |
| SHA256 | 2^64 |
| SHA512/224 | 2^128 |
| SHA512/256 | 2^128 |
| SHA384 | 2^128 |
| SHA512 | 2^128 |

根据填充规则`MD5` `SHA1` `SHA224` `SHA256`哈希算法在数据的末尾提供了`[8]byte`的空间用于存储所处理数据的`bit`数，这样最大处理长度就是`2^64`个`bit`，超出的话填充长度部分会出现溢出问题，从而会因为安全风险。

而`SHA512/224` `SHA512/256` `SHA384` `SHA512`哈希算法的填充规则提供了`[16]byte`的空间用于存储长度，这样最大处理长度就是`2^128`个`bit`，超出的话也会有同样的风险

### 填充方式在Golang中的问题

1. `Golang`中用一个`uint64`变量来记录处理的`字节数`，在生成填充数据的时候需要乘以`8`转换成`bit`数，如果记录的`字节数`超过了`uint64`最大值的`1/8`就会产生溢出
  
2. `Golang`中处理`SHA512/224` `SHA512/256` `SHA384` `SHA512`系列方法时，同样用的`uint64`记录处理的`字节数`，除了上述问题之外，`uint64`记录的`字节数`小于规定的最大值`2^128`个`bit`，也就是说`Golang`中的`SHA512`系列方法，只能处理`2^64`个`bit`，远比规定的极限值要小很多

## 基于时间的侧信道攻击 side channel attack

`侧信道攻击`是一种利用计算机不经意间释放出的信息信号（如电磁辐射，电脑硬件运行声）来进行破译的攻击模式：例如，黑客可以通过计算机显示屏或硬盘驱动器所产生的电磁辐射，来读取你所显示的画面和磁盘内的文件信息；或是，通过计算机组件在执行某些程序时需要消耗不同的电量，来监控你的电脑；亦或是，仅通过键盘的敲击声就能知道你的账号和密码。

签名验证过程中会被大量用到的就是签名字符串判等，长度相等的两个字符串判等过程中，从第一个不等的字符开始就不需要再比较了，后面肯定是不等的。也就是从第一个不等字符开始就判定为不等，直接返回了。这样全等字符串判定和不等字符串判定的处理时间就会有差异，这样就有可能被利用，触发基于时间的`侧信道攻击`

对应的解决方式就是`subtle`包，`subtle`包中方法经常被用于加密相关代码，但是官方提示要**谨慎使用**

{% code title="使用 subtle.ConstantTimeCompare 判等示例" %}
```go
subtle.ConstantTimeCompare(byteSlice1, byteSlice2) == 1 //全等为true
```
{% endcode %}

## 长度扩展攻击

在日常的开发中，很有可能会开发一些接口给第三方使用，为了保证数据在传输过程中不被篡改，一般会考虑对请求参数做`签名验证`

对数据做哈希签名只能解决数据`完整性`问题，但不能确认是否是真实第三方发出的，为了解决身份认证的问题，会在签名前的数据中加入第三方和服务方共有的`机密数据`，这样服务端收到数据后既能验证第三方的身份又能保证数据的完整性

{% hint style="warning" %}
这种`机密数据`放在前面的就会有`长度扩展攻击`风险
```text
hash(机密数据 + 请求参数)
```
{% endhint %}

## 攻击构造原理
![哈希长度扩展攻击示意图](../images/hash/hash-length-extension-attack.png)

- 图中灰色部分是明文传输，属于已知部分
- 橙色部分为`机密数据`，内容未知，但是我们并不需要内容，而只需要`机密数据`的长度，这部分数据的长度可以根据服务端文档进行猜测
- 黄色的`填充`数据，规则已知，可以自行构造

攻击数据构造就是根据`请求参数`及明文传输的`摘要值`（图中灰色部分），构造该摘要算法计算完某个`BlockSize`之后的中间状态，继续写入`攻击数据`进行新一轮的迭代，生成`新摘要`，并将新生成的带攻击数据的参数和新摘要替换原有数据进行请求攻击

## 示例
### 签名方式说明
假定现在有一个批量获取订单数据的接口，采用`AppId` `AppKey`的模式签名，具体方式如下

- 从服务方获得`AppId` `AppKey`
- 请求参数`appid=xxxxx` `orderids=xxxxxx,xxxxx`，按照key升序排列拼接成待签名字符串

```text
appid=xxxxx&orderids=xxxxxx,xxxxx
```

- 请求时构造签名参数，规则如下：

```text
sign = hash(AppKey + appid=xxxxx&orderids=xxxxxx,xxxxx)

//生成的请求链接为
xxx.com?appid=xxxxx&orderids=xxxxxx,xxxxx&sign=sign
```

伪代码中`hash`方法包括，但不限于`MD5` `SHA1` `SHA2系列`等，之所以单独说了这几种哈希算法是因为他们都是基于`Merkle–Damgård`构造模式进行哈希计算的，具体的流程可以翻看前面的示意图

### 配套代码

{% tabs %}
{% tab title="MD5" %}
```go
const KEY = "43f55d8396d0982fd62666883a9b3730"
data := "appid=1&orderIds=11111,999"
injectData := ",22222,33333"

md5Sign := md5.Sum([]byte(KEY + data))
fmt.Println("origin md5:", hex.EncodeToString(md5Sign[:]))
fmt.Println("origin query:", data)

b := NewBuilder([]byte(data), md5Sign[:], len(KEY), crypto.MD5)
padding, newMD5, err := b.Build()
if err != nil {
    t.Error(err)
}

newMD5.Write([]byte(injectData))
newSign := newMD5.Sum(nil)
fmt.Println("new md5: ", hex.EncodeToString(newSign))
fmt.Println("new query: ", data+string(padding)+injectData)

if checkSum(newSign, []byte(KEY+data+string(padding)+injectData), crypto.MD5) {
    fmt.Println("It works for " + crypto.MD5.String())
}
```
{% endtab %}

{% tab title="SHA1" %}
```go
const KEY = "43f55d8396d0982fd62666883a9b3730"
data := "appid=1&orderIds=11111,999"
injectData := ",22222,33333"

sha1Sign := sha1.Sum([]byte(KEY + data))
fmt.Println("origin sha1:", hex.EncodeToString(sha1Sign[:]))
fmt.Println("origin query:", data)

b := NewBuilder([]byte(data), sha1Sign[:], len(KEY), crypto.SHA1)
padding, newSha1, err := b.Build()
if err != nil {
    t.Error(err)
}

newSha1.Write([]byte(injectData))
newSign := newSha1.Sum(nil)
fmt.Println("new sha1: ", hex.EncodeToString(newSign))
fmt.Println("new query: ", data+string(padding)+injectData)

if checkSum(newSign, []byte(KEY+data+string(padding)+injectData), crypto.SHA1) {
    fmt.Println("It works for " + crypto.SHA1.String())
}
```
{% endtab %}

{% tab title="SHA256" %}
```go
const KEY = "43f55d8396d0982fd62666883a9b3730"
data := "appid=1&orderIds=11111,999"
injectData := ",22222,33333"

sha256Sign := sha256.Sum256([]byte(KEY + data))
fmt.Println("origin sha256:", hex.EncodeToString(sha256Sign[:]))
fmt.Println("origin query:", data)

b := NewBuilder([]byte(data), sha256Sign[:], len(KEY), crypto.SHA256)
padding, newSha256, err := b.Build()
if err != nil {
    t.Error(err)
}

newSha256.Write([]byte(injectData))
newSign := newSha256.Sum(nil)
fmt.Println("new sha256: ", hex.EncodeToString(newSign))
fmt.Println("new query: ", data+string(padding)+injectData)

if checkSum(newSign, []byte(KEY+data+string(padding)+injectData), crypto.SHA256) {
    fmt.Println("It works for " + crypto.SHA256.String())
}
```
{% endtab %}

{% tab title="SHA512" %}
```go
const KEY = "43f55d8396d0982fd62666883a9b3730"
data := "appid=1&orderIds=11111,999"
injectData := ",22222,33333"

sha512Sign := sha512.Sum512([]byte(KEY + data))
fmt.Println("origin sha512:", hex.EncodeToString(sha512Sign[:]))
fmt.Println("origin query:", data)

b := NewBuilder([]byte(data), sha512Sign[:], len(KEY), crypto.SHA512)
padding, newSha512, err := b.Build()
if err != nil {
    t.Error(err)
}

newSha512.Write([]byte(injectData))
newSign := newSha512.Sum(nil)
fmt.Println("new sha512: ", hex.EncodeToString(newSign))
fmt.Println("new query: ", data+string(padding)+injectData)

if checkSum(newSign, []byte(KEY+data+string(padding)+injectData), crypto.SHA512) {
    fmt.Println("It works for " + crypto.SHA512.String())
}
```
{% endtab %}
{% endtabs %}

`MD5` `SHA1` `SHA256` `SHA512`会完全命中上面所说的问题。

相对的，像`SHA224` `SHA384` `SHA512/224` `SHA512/256`由于返回值做了截取，如果发动`扩展长度攻击`需要穷举大量的数据，成功率不高

{% hint style="info" %}
详细代码实现参见 [hashpump](https://github.com/coredumptoday/hashpump)

支持 `MD5` `SHA1` `SHA256` `SHA512` 攻击参数构造
{% endhint %}

{% hint style="success" %}
**Hash-based Message Authentication Code**

相对优雅的解决`身份+完整性`的验证，请使用`hmac`，基于Hash函数和密钥进行消息认证的方法
{% endhint %}
