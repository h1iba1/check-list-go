## 函数api

http库相关的SSRF攻击函数：

```go
func Get(url string) (resp *Response, err error)：直接请求url

func Post(url, contentType string, body io.Reader) (resp *Response, err error)：发送POST请求，可自定义contentType和body

func PostForm(url string, data url.Values) (resp *Response, err error)：发送POST请求，只能发送form形式的请求体

func Head(url string) (resp *Response, err error)：对url发送head请求

func NewRequest(method string, url string, body string)：定制化请求
```

beego相关SSRF攻击api：

| API                                                          | 功能                                            | 可能存在的隐患                                               |
| ------------------------------------------------------------ | ----------------------------------------------- | ------------------------------------------------------------ |
| httplib.Get、httplib.Post、httplib.Put、httplib.Delete、httplib.Head | 定义对应Method的请求BeegoHTTPRequest，参数是url | URL可篡改的SSRF攻击                                          |
| BeegoHTTPRequest.Header                                      | 定义BeegoHTTPRequest的请求头                    | 请求头可篡改的SSRF攻击                                       |
| BeegoHTTPRequest.SetCookie                                   | 定义BeegoHTTPRequest的Cookie                    | Cookie可篡改的SSRF攻击                                       |
| BeegoHTTPRequest.String、BeegoHTTPRequest.Bytes、BeegoHTTPRequest.DoRequest | 发送对应的HTTP请求                              | 需要查看请求各个参数的设定是否可控，如果可控，则存在SSRF漏洞 |





## 检测

- go中http请求的三方包非常多，可以通过搜索`http.|httplib.|.Get(|.Post(`等关键字定位到相关的http请求代码，查看上下文判断请求的url外部是否可控。

