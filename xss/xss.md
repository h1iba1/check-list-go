go通常用作开发后端，与前端交互的形式主要是作为api提供给前端，因此此类问题较少。但go也完全支持渲染html模板，如果使用了go内置的模板渲染功能，可能还是存在xss问题。

go 内置了html、text/template和html/template三种html旋绕包，它们的区别：

- html (需要使用EscapeString来转义) https://golang.org/pkg/html/#pkg-overview 【不推荐使用】 仅仅对这五个字符进行编码` <, >, &, ' and ".`对一些代码注入和模板注入等无能为力。
- text/template（html需要使用HTMLEscapeString来转义，js需要JSEscapeString来转义） https://golang.org/pkg/text/template/#HTMLEscapeString
- html/template（html需要HTMLEscapeString来转义，js需要JSEscapeString来转义） https://golang.org/pkg/html/template/#HTMLEscapeString



## 检测

- 搜索代码中`html|text/template|html/template`，查看参数是否可控和做了过滤
- go内置html渲染的数据的场景来子用户的输入或者外部调用的参数等，所以可以通过`req.FormValue|req.Form.Get|r.Form|req.getParameter|http.ResponseWriter`等关键字来识别出这些场景。





## 参考

https://github.com/he1m4n6a/Go_Security_Study/tree/master/%E7%BC%96%E7%A0%81%E5%AE%89%E5%85%A8/WEB%E5%AE%89%E5%85%A8#web