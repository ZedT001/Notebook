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
/invokefunction：可以简单理解为一个被定义的方法，但是由于写了static(静态)，所以可以在不通过new实例化的情况下调用

源码长这样：public static function invokeFunction($function, $vars = [])

function=call_user_func_array：call：调用执行、user_func：用户定义的函数、array：数组

翻译成人话就是调用一个用户函数，并把数组作为它的参数传进去，通过function来传参

&vars[0]=phpinfo&vars[1][]=1：因为在源码中 'vars' 长这样：" $vars = [] "，所以是可以通过数组形式进行传参的

## 为什么框架会被骗？
ThinkPHP 的底层代码在解析 URL 时，本来以为只传个普通的办公室名字(就类似于passport)，结果一看你传了个带反斜杠的 \think\app。由于早期版本没有对反斜杠做严格的过滤，框架真的把你带到了底层的“总控制室”，并且让你按下了执行按钮。

这就是为什么前面必须要有 /（用来分割模块和方法），中间要有 \（用来指定底层核心类）。

## 补充点1：URL 里明明传的是普通文本 vars[0]=phpinfo，服务器是怎么把它变成 PHP 数组的？
原理：在 PHP 中，当你通过 GET 或 POST 传递参数时，如果参数名中包含中括号 []，PHP 底层（Zend 引擎）在解析时，会自动把它转换成数组。

$vars[0] 会被解析成：$vars = array(0 => 'phpinfo')

这就是为什么在 URL 里写方括号，后端就能收到数组

## 补充点2：为什么一个带 []，另一个不带？（一维与二维的区别）

仔细看写的 Payload：&vars[0]=phpinfo&vars[1][]=1

vars[0]：指定了键名为 0，没有后面的 []，它就是普通的字符串。

vars[1][]：为什么这里有两个方括号？因为 call_user_func_array 要求的格式是 [函数名, [参数1, 参数2...]]。第二个参数必须又是一个数组（存放多个参数的列表）。所以 vars[1][] 的意思是：“在 vars 数组的 1 号位置，放一个子数组，子数组里面装数字 1”

## 补充点3：如果源码不是 []，而是字符串会怎样？

假设源码写的是 public function invokeFunction($function, $vars = '')（默认是字符串，而不是数组）。

这时候，你在 URL 传 vars[0]=phpinfo，后端框架接收到的 $vars 就是一个数组。此时底层的 bindParams 函数如果直接把它当字符串处理，就会直接报错：Array to string conversion（数组无法转换为字符串），攻击直接失败。所以，“能传数组”的前提，一定是源码里期望接收或者默认接收数组。

## 拓展：关于数组传参的理解
1.源码中 $vars = [] 表明该参数期望接收一个数组。

2.源码中 $vars = [] 表明该参数期望接收一个数组。

3.双重方括号 [][] 可以构造多维数组，完美契合了 call_user_func_array 需要“函数名+参数数组”的复合结构。

4.双重方括号 [][] 可以构造多维数组，完美契合了 call_user_func_array 需要“函数名+参数数组”的复合结构。
