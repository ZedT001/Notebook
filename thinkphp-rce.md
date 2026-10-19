markdown
# ThinkPHP 5.0.22 RCE 漏洞手工复现

## 1. 漏洞背景
在使用黑盒测试时，通过指纹识别发现目标使用了 ThinkPHP 5.0.22 框架，该版本存在远程代码执行漏洞。
### （url栏出现了'/index.php?s='被认定为thinkphp专属标识）

## 2. 环境搭建
- 本地使用 phpStudy 搭建环境。
- PHP 版本必须降至 5.6 或 7.0（高版本 7.4 会因反射机制报错）。
- 网站根目录必须指向 `public` 文件夹。

## 3. 验证漏洞
使用最无害的 `phpinfo` 函数进行验证，手工构造的 Payload 如下：

```http
GET /index.php?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=phpinfo&vars[1][]=1 HTTP/1.1
```
## Payload详解：
thinkphp框架的结构为【模块/控制器/方法】用 '/' 正斜杠隔开

eg：index.php?s=store/passport/login

翻译成人话就是在store这个模块下找passport这个控制器执行login方法，

index/：代表在index这个模块下没问题，本来按照thinkphp框架的结构后面应该跟的是【控制器/方法】这种形式，但是！在thinkphp5下【\think\app】是系统核心底层的隐藏目录，【\think\app】是框架内部最核心的一个类
/invokefunction：可以简单理解为一个被定义的类，比如class student，invokefunction在这段代码中的作用等价于student，都是通过声明后来进行调用这个类

源码长这样：public static function invokeFunction($function, $vars = [])

## 为什么框架会被骗？
ThinkPHP 的底层代码在解析 URL 时，本来以为你要传个普通的办公室名字(就类似于passport)，结果一看你传了个带反斜杠的 \think\app。由于早期版本没有对反斜杠做严格的过滤，框架真的把你带到了底层的“总控制室”，并且让你按下了执行按钮。

这就是为什么前面必须要有 /（用来分割模块和方法），中间要有 \（用来指定底层核心类）。
