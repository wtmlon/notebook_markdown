# handler接口

```go
type Handler interface{
    ServeHTTP(ResponseWriter, *Request)
}
```

普通路由写法

```go
    http.HandleFunc("/", mainHandler)
    http.HandleFunc("/hello/", helloHandler)
    http.HandleFunc("/world/", worldHandler)

    server := http.Server{
        Addr: ":8080",
    }
    server.ListenAndServe()
```

此时访问：

- localhost:8080/hello：会响应helloHandler函数
- localhost:8080/hello/：同样会响应helloHandler函数

实际上，访问`localhost:8080/hello`时，其实会以301重定向方式自动补齐为：`/hello/`，然后浏览器自动发起第二次请求。

如果使用ServeMux注册路由：

```go
    mux := http.NewServeMux()
    mux.HandleFunc("/", mainHandler)
    mux.HandleFunc("/hello", helloHandler)
    mux.HandleFunc("/world", worldHandler)

    server := http.Server{
        Addr: ":8080",
        Handler: mux,
    }
    server.ListenAndServe()
```

此时访问：

- localhost:8080/hello：会响应helloHandler函数
- localhost:8080/hello/：会响应mainHandler函数！！！

在使用ServeMux时：

- 如果pattern以"/"开头，表示匹配URL的路径部分
- 如果pattern不以"/"开头，表示从host开始匹配
- 匹配时长匹配优先于短匹配，注册在"/"上的pattern会被所有请求匹配，但其匹配长度最短
- 如果pattern带上了尾随斜线"/"，ServeMux将会对请求不带尾随斜线的URL进行301重定向。例如，在"/images/"模式上注册了一个handler，当请求的URL路径为"/images"时，将自动重定向为"/images/"，除非再单独为"/images"模式注册一个handler。
- 如果为"/images"注册了handler，当请求URL路径为"/images/"时，将无法匹配到.

# 中间件写法

中间件的作用是在执行对应handler的前后进行请求相关校验操作

```go
package main


import(
    "fmt"
    "net/http"
)

func before(handle http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r * http.Request) {
        fmt.Println("执行前置处理")
        handle(w, r)
    }
}

func test(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "test1")
}


func main() {

    http.HandleFunc("/", before(test))

    server := http.Server{
        Addr: "127.0.0.1:8080",
    }
    server.ListenAndServe()

}
```

# json序列化使用

需要import "encoding/json"

```go
    type Person struct {
        Name string
        Age int
    }

    p := Person {
        Name: "lisi",
        Age: 50,
    }

    data, _ := json.Marshal(&p)
    fmt.Printf(string(data));         //{"Name":"lisi","Age":50}
```

作用是达到go map||struct到json字符串之间的相互转化。

## go map中tag定义

在结构体序列化时，如果希望序列化后的`key`的名字可以自定义，可以给该结构体指定一个`tag`标签：

```go
    type Person struct {
        Name string `json:"my_name"`
        Age int `json:"my_age"`
    }
    //序列化的结果：{"my_name":"lisi","my_age":50}
```

在定义`struct tag`的时候需要注意的几点是:

- 字段的tag是`"-"`，那么这个字段不会输出到JSON
- tag中如果带有`"omitempty"`选项，那么如果该字段值为空，就不会输出到JSON串中
- 如果字段类型是bool, string, int, int64等，而tag中带有`",string"`选项，那么这个字段在输出到JSON的时候会把该字段对应的值转换成JSON字符串
- JSON对象只支持string作为key，所以要编码一个map，那么必须是map[string]T这种类型(T是Go语言中任意的类型)
- Channel, complex和function是不能被编码成JSON的
- 嵌套的数据是不能编码的，不然会让JSON编码进入死循环
- 指针在编码的时候会输出指针指向的内容，而空指针会输出null

## json反序列化

```go
    str :=  `{"Name":"lisi","Age":50}`

    // 反序列化json为结构体
    type Person struct {
        Name string 
        Age int 
    }

    var p Person
    json.Unmarshal([]byte(str), &p)
    fmt.Println(p)                            //{lisi 50}
```

## json反序列化到未知类型（interface{})

我们知道json的数据结构，可以直接进行序列化操作，如果不知道JSON具体的结构，就需要解析到interface，因为interface{}可以用来存储任意数据类型的对象。

JSON包中采用`map[string]interface{}`和`[]interface{}`结构来存储任意的JSON对象和数组。Go类型和JSON类型的对应关系如下：

- bool 代表 JSON booleans,
- float64 代表 JSON numbers,
- string 代表 JSON strings,
- nil 代表 JSON null

```go
    jsonStr := `{"Name":"Lisi","Age":6,"Parents":["Lisan","WW"]}`
    jsonBytes := []byte(jsonStr) 

    var i interface{}
    json.Unmarshal(jsonBytes, &i)
    fmt.Println(i)        // map[Age:6 Name:Lisi Parents:[Lisan WW]]
```

记得从json转map要先转成字节流[]byte

上述变量`i`存储了存储了一个map类型，key是strig，值存储在空接口内， 如果在我们不知道他的结构的情况下，我们把他解析到interface{}里面，其真实结构如下：

```go
i = map[string]interface{}{
    "Name": "Lisi",
    "Age":  6,
    "Parents": []interface{}{
        "Lisan",
        "WW",
    },
}
```

[]为未知类型组成的slice， 遍历时，不知道具体数值type所以要用类型断言。

```go
    m := i.(map[string]interface{})
    for k, v := range m {
        switch r := v.(type) {
        case string:
            fmt.Println(k, " is string ", r)
        case int:
            fmt.Println(k, " is int ", r)
        case []interface{}:
            fmt.Println(k, " is array ", )
            for i, u := range r {
                fmt.Println(i, u)
            }
        default:
            fmt.Println(k, " cannot be recognized")
        }
    }
```

上面是官方提供的解决方案，操作起来不是很方便，推荐使用第三方包有：

- [GitHub - bitly/go-simplejson: a Go package to interact with arbitrary JSON](https://github.com/bitly/go-simplejson)
- [GitHub - thedevsaddam/gojsonq: A simple Go package to Query over JSON/YAML/XML/CSV Data](https://github.com/thedevsaddam/gojsonq)

# XML文件解析

xml文件如下

```xml
<?xml version="1.0" encoding="utf-8"?>
<books version="1">
    <book>
        <bookName>离散数学</bookName>
        <bookPrice>120</bookPrice>
    </book>
    <book>
        <bookName>人月神话</bookName>
        <bookPrice>75</bookPrice>
    </book>
</books>
```

## 反序列化

```go
package main

import (
    "encoding/xml"
    "fmt"
    "io/ioutil"
    "os"
)

type BookStore struct {
    XMLName     xml.Name `xml:"books"`
    Version     string   `xml:"version,attr"`
    Store       []book     `xml:"book"`
    Description string   `xml:",innerxml"`
}

type book struct {
    XMLName        xml.Name `xml:"book"`
    BookName     string   `xml:"bookName"`
    BookPrice   string   `xml:"bookPrice"`
}

func main() {

    file, err := os.Open("books.xml")         
    if err != nil {
        fmt.Printf("error: %v", err)
        return
    }
    defer file.Close()
    data, err := ioutil.ReadAll(file)
    if err != nil {
        fmt.Printf("error: %v", err)
        return
    }

    v := BookStore{}
    err = xml.Unmarshal(data, &v)
    if err != nil {
        fmt.Printf("error: %v", err)
        return
    }

    fmt.Println(v)
}
```

# 从Request中获取Form字段

Request的Form字段包括Get请求中的url参数，和Post请求中的请求体json内容。

```go
if len(r.Form["username"][0])==0{
    //为空的处理
}
```

`r.Form`对不同类型的表单元素的留空有不同的处理：

- 空文本框、空文本区域以及文件上传，元素的值为空值
- 未选中的复选框和单选按钮，则不会在r.Form中产生相应条目，如果我们用上面例子中的方式去获取数据时程序就会报错。所以我们需要通过`r.Form.Get()`来获取值，因为如果字段不存在，通过该方式获取的是空值。但是通过`r.Form.Get()`只能获取单个的值，

# 文件上传处理

上传html

```html
<html>
<head>
    <title>上传文件</title>
</head>
<body>
<form enctype="multipart/form-data" action="/upload" method="post">
  <input type="file" name="uploadfile" />
  <input type="submit" value="upload" />
</form>
</body>
</html>
```

form的`enctype`属性有如下三种情况:

```
application/x-www-form-urlencoded        # 使用url编码，服务端需要相应解码
multipart/form-data                          # 文件上传使用，不会不对字符编码。
text/plain                                  # 空格转换为 "+" 加号，但不对特殊字符编码。
```

go后端处理代码

```go
// 上传文件处理路由：http.HandleFunc("/upload", upload)
func upload(w http.ResponseWriter, r *http.Request) {

    // 设置上传文件能使用的内存大小，超过了，则存储在系统临时文件中
    r.ParseMultipartForm(32 << 20)
    // 获取上传文件句柄
    file, handler, err := r.FormFile("uploadfile")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer file.Close()

    fmt.Fprintf(w, "%v", handler.Header)
    f, err := os.OpenFile("./upload/" + handler.Filename, os.O_WRONLY|os.O_CREATE, 0666)
    if err != nil {
        fmt.Println(err)
        return
    }
    defer f.Close()
    io.Copy(f, file)
}
```

# 表单防止重复提交

防止表单重复提交的方案有很多，其中之一是在表单中添加一个带有唯一值的隐藏字段：

- 1.在服务器端生成一个唯一的随机标识号，专业术语称为Token(令牌)，同时在当前用户的Session域中保存这个Token。
- 2.将Token发送到客户端的Form表单中，在Form表单中使用隐藏域来存储这个Token
- 3.表单提交的时候连同这个Token一起提交到服务器端，然后在服务器端判断客户端提交上来的Token与服务器端生成的Token是否一致，如果不一致，那就是重复提交了，此时服务器端就可以不处理重复提交的表单。如果相同则处理表单提交，处理完后清除当前用户的Session域中存储的标识号。

在下列情况下，服务器程序将拒绝处理用户提交的表单请求：

- 存储Session域中的Token(令牌)与表单提交的Token(令牌)不同。
- 当前用户的Session中不存在Token(令牌)。
- 用户提交的表单数据中没有Token(令牌)。

```html
用户名:<input type="text" name="username">
密码:<input type="password" name="password">
<input type="hidden" name="token" value="{{.}}">
<input type="submit" value="登陆">
```

在模版里面增加了一个隐藏字段`token`，该值通过MD5(时间戳)来确定唯一值，然后我们把这个值存储到服务器端，以方便表单提交时比对判定。

```go
func login(w http.ResponseWriter, r *http.Request) {    r.ParseForm()    token := r.Form.Get("token")        if token != "" {        //验证token的合法性
    } else {        //不存在token报错
    }        // 执行具体登录业务
}
```

# 鉴权操作

在网站中，有些页面是登录后的用户才能访问的，由于http是无状态的协议，我们无法确认用户的状态（如是否登录）。这时候浏览器在访问这些页面时，需要额外传输一些用户的账户信息给后台，让后台知道该用户是否登录、是哪个用户在访问。

## cookie

cookie是浏览器实现的技术，在浏览器中可以存储用户是否登录的凭证，每次请求都会将该凭证发送给服务器。

cookie实现鉴权步骤：

- 用户登录成功后，后端向浏览器设置一个cookie：username=lisi
- 每次请求，浏览器会自动把该cookie发送给服务端
- 服务端处理请求时，从cookie中取出username，就知道是哪个用户了
- 如果没过期，则鉴权通过，过期了，则重定向到登录页

```go
// 登录时设置cookie
expiration := time.Now()
expiration = expiration.AddDate(1, 0, 0)
cookie := http.Cookie{Name: "username", Value: "张三", Expires: expiration}
http.SetCookie(w, &cookie)

// 再次访问时，获取浏览器传递的cookie
// 获取cookie方式一，下面的r为Request struct
username, _ := r.Cookie("username")
// 获取cookie方式二
for _, cookie := range r.Cookies() {
	fmt.Println(cookie.Username)
}
```

容易看出是明文传输的，且依靠上传的cookie判断超时，所以可以被轻易伪造

## session方式

为了解决cookie的安全问题，基于cookie，衍生了session技术。session技术将用户的信息存储在了服务端，保证了安全，其实现步骤为：

- 服务端设置cookie时，不再存储username，而是存储一个随机生成的字符串，比如32位的uuid，服务端额外存储一个uuid与用户名的映射
- 用户再次请求时，会自动把cookie中的uuid带入给服务器
- 服务器使用uuid进行鉴权

一般上述的uuid在cookie中存储的键都是sid（session_id），也就是常说的session方案，服务端此时需要额外开辟空间存储sid与用户真实信息的对应映射。

如果要手动实现session，需要注意以下方面：

- 全局session管理器：
- 保证sessionid 的全局唯一性
- 为每个客户关联一个session
- session 过期处理
- session 的存储(可以存储到内存、文件、数据库等)

关于session数据（sid与真实用户的映射）的存储，可以存放在服务端的一个文件中，比如该session第三方库：https://github.com/gorilla/sessions

```go
package main

import(
	"fmt"
    "net/http"
    "github.com/gorilla/sessions"
)

// 利用cookie方式创建session，秘钥为 mykey
var store = sessions.NewCookieStore([]byte("mykey"))

func setSession(w http.ResponseWriter, r *http.Request){
    session, _ := store.Get(r, "sid")
    session.Values["username"] = "张三"
    session.Save(r, w)
}

func profile(w http.ResponseWriter, r *http.Request){

    session, _ := store.Get(r, "sid")

    if session.Values["username"] == nil {
        fmt.Fprintf(w, `未登录，请前往 localhost:8080/setSession`)
        return
    }

    fmt.Fprintf(w, `已登录，用户是：%s`, session.Values["username"])
    return
}

func main() {

    // 访问隐私页面
    http.HandleFunc("/profile", profile)

    // 设置session
    http.HandleFunc("/setSession", setSession)

    server := http.Server{
        Addr: ":8080",
    }
	server.ListenAndServe()

}
```

在企业级开发中，经常使用额外的数据库redis来存储session数据。

    以上方式中，生成的sid都存储在cookie中，如果用户禁用了cookie，则每次请求服务端无法收到sid！我们需要想别的办法来让浏览器的每次请求都携带上sid，常用方式是URL重写：在返回给用户的页面里的所有的URL后面追加session标识符，这样用户在收到响应之后，无论点击响应页面里的哪个链接或提交表单，都会自动带上session标识符，从而就实现了会话的保持。

## JWT鉴权

session将数据存储在了服务端，无端造成了服务端空间浪费，可否像cookie那样将用户数据存储在客户端，而不被黑客破解到呢？

JWT是json web token缩写，它将用户信息加密到token里，服务器不保存任何用户信息。服务器通过使用保存的密钥验证token的正确性，只要正确即通过验证。 JWT和session有所不同，session需要在服务器端生成，服务器保存session,只返回给客户端sessionid，客户端下次请求时带上sessionid即可。因session是储存在服务器中，有多台服务器时会出现一些麻烦，需要同步多台主机的信息，不然会出现在请求A服务器时能获取信息，但是请求B服务器身份信息无法通过。JWT能很好的解决这个问题，服务器端不用保存jwt，只需要保存加密用的secret，在用户登录时将jwt加密生成并发送给客户端，由客户端存储，以后客户端的请求带上，由服务器解析jwt并验证。这样服务器不用浪费空间去存储登录信息，也不用浪费时间去做同步。

### jwt构成

一个 JWT token包含3部分:

- header: 告诉我们使用的算法和 token 类型
- Payload: 必须使用 sub key 来指定用户 ID, 还可以包括其他信息比如 email, username 等.
- Signature: 用来保证 JWT 的真实性. 可以使用不同算法

header:

```
// base64编码的字符串`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`
// 这里规定了加密算法,hash256
{
  "alg": "HS256",
  "typ": "JWT"
}
```

payload：

```
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
// base64编码的字符串`eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9`
```

这里的内容没有强制要求,因为 Payload 就是为了承载内容而存在的,不过想用规范的话也可以参考下面的：

```
* iss: jwt签发者
* sub: jwt所面向的用户
* aud: 接收jwt的一方
* exp: jwt的过期时间，这个过期时间必须要大于签发时间
* nbf: 定义在什么时间之前，该jwt都是不可用的.
* iat: jwt的签发时间
* jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。
```

signature是用 header + payload + secret组合起来加密的,公式是:

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

这里 secret就是自己定义的一个随机字符串,这一个过程只能发生在 server 端，会随机生成一个 hash 值，这样组合起来之后就是一个完整的 jwt 了:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.4c9540f793ab33b13670169bdf444c1eb1c37047f18e861981e14e34587b1e04
```

### jwt执行流程

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-12-15-26-58-image.png)

#### Go使用jwt

整体操作步骤：

- 1.从request获取tokenstring
- 2.将tokenstring转化为未解密的token对象
- 3.将未解密的token对象解密得到解密后的token对象
- 4.从解密后的token对象里取参数

使用第三方包：`go get github.com/dgrijalva/jwt-go` 示例：

```go
// 生成Token：
// SecretKey 是一个 const 常量
func CreateToken(SecretKey []byte, issuer string, Uid uint, isAdmin bool) (tokenString string, err error) {      claims := &jwtCustomClaims{         jwt.StandardClaims{           ExpiresAt: int64(time.Now().Add(time.Hour * 72).Unix()),             Issuer:    issuer,         },         Uid,         isAdmin,     }     token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)     tokenString, err = token.SignedString(SecretKey)     return  }

// 解析Token
func ParseToken(tokenSrt string, SecretKey []byte) (claims jwt.Claims, err error) {     var token *jwt.Token     token, err = jwt.Parse(tokenSrt, func(*jwt.Token) (interface{}, error) {        return SecretKey, nil     })     claims = token.Claims     return  }
```
