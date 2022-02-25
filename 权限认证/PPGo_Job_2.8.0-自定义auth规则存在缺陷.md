## 1.自定义auth规则存在缺陷

### 实列：

项目地址：https://github.com/george518/PPGo_Job

PPGo_Job 2.8.0 权限绕过。

### 漏洞原因：

自定义auth规则存在缺陷。

```go
func (self *BaseController) Auth() {
	arr := strings.Split(self.Ctx.GetCookie("auth"), "|")
	self.userId = 0
	// auth第一个值大于1
	if len(arr) == 2 {
		idstr, password := arr[0], arr[1]
		userId, _ := strconv.Atoi(idstr)
		if userId > 0 {
			user, err := models.AdminGetById(userId)

			if err == nil && password == libs.Md5([]byte(self.getClientIp()+"|"+user.Password+user.Salt)) {
				self.userId = user.Id
				self.loginName = user.LoginName
				self.userName = user.RealName
				self.user = user
				self.AdminAuth()
				self.dataAuth(user)
			}

			isHasAuth := strings.Contains(self.allowUrl, self.controllerName+"/"+self.actionName)
			noAuth := "ajaxsave/table/loginin/loginout/getnodes/start／apitask/apistart/apipause"
			isNoAuth := strings.Contains(noAuth, self.actionName)

			// 能走到这里。并且
			if isHasAuth == false && isNoAuth == false {
				if strings.Contains(self.actionName, "ajax") {
					self.ajaxMsg("没有权限", MSG_ERR)
					return
				}

				flash := beego.NewFlash()
				flash.Error("没有权限")
				flash.Store(&self.Controller)
				return
			}
		}
	}

	if self.userId == 0 &&
		(self.controllerName != "login" &&
			self.actionName != "loginin" &&
			self.actionName != "apistart" &&
			self.actionName != "apitask" &&
			self.actionName != "apipause" &&
			self.actionName != "apisave" &&
			self.actionName != "apistatus" &&
			self.actionName != "apiget") {
		self.redirect(beego.URLFor("LoginController.Login"))
	}
}
```

当auth >1时。

进入if userId > 0 {}。

User查询为空。

```go
isHasAuth := strings.Contains(self.allowUrl, self.controllerName+"/"+self.actionName)
noAuth := "ajaxsave/table/loginin/loginout/getnodes/start／apitask/apistart/apipause"
isNoAuth := strings.Contains(noAuth, self.actionName)

if isHasAuth == false && isNoAuth == false {
				if strings.Contains(self.actionName, "ajax") {
					self.ajaxMsg("没有权限", MSG_ERR)
					return
				}

				flash := beego.NewFlash()
				flash.Error("没有权限")
				flash.Store(&self.Controller)
				return
			}
		}
```

self.actionName为空。

```go
isNoAuth := strings.Contains(noAuth, self.actionName)==

fmt.Println(strings.Contains("ajaxsave/table/loginin/loginout/getnodes/start／apitask/apistart/apipause",""))
```

Strings.Contains()第二个值，为空则结果无论如何都为true.

isNoAuth为true则绕过下面的鉴权。self.userId不为0则绕过最下面的鉴权。达到绕过的目的。



## 参考链接：

https://forum.butian.net/share/928