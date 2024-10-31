---
title: PHP 完善的 Error / Exception 的捕获与处理
categories: [后端, PHP]
tags: [PHP]
---

PHP（PHP_VERSION >= 7） 的 Error / Exception 的捕获与处理还是值得一说的，优雅处理错误与异常，在提升框架友好度的同时，也提升了开发效率。

## PHP 错误等级

```ini
# 系统级用户代码的一些错误类型 可由 try ... catch ... 捕获
E_PARSE          解析时错误 语法解析错误 少个分号 多个逗号一类的 致命错误
E_ERROR          运行时错误 比如调用了未定义的函数或方法 致命错误

# 系统级用户代码的一些错误类型 可由 set_error_handler 捕获处理
E_WARNING        运行时警告 调用了未定义的变量
E_NOTICE         运行时提醒
E_DEPRECATED     运行时已废弃的函数或方法

# 用户级自定义错误 可由 trigger_error 触发 可由 set_error_handler 捕获处理
E_USER_ERROR      用户自定义错误 致命错误 未处理也会导致程序退出
E_USER_WARNING
E_USER_NOTICE
E_USER_DEPRECATED

==========================开发中常遇到/不常遇到分割线=======================

# Zend Engine 内部的一些错误 应该也能通过 try ... catch ... 捕获 略难测试
E_CORE_ERROR
E_CORE_WARNING
E_COMPILE_ERROR
E_COMPILE_WARNING

#编码标准化警告(建议如何修改以向前兼容)
E_STRICT          部分 try ... catch ... 部分 set_error_handler
E_RECOVERABLE_ERROR
```

以上为 PHP 的一些错误监听级别，常用于 error_reporting 和 set_error_handler 的监听级别设定。

```php
<?php
error_reporting(E_ALL & ~E_NOTICE);

set_error_handler(function handler($error_no, $error_msg, $error_file, $error_line) {
}, E_ALL | E_STRICT);
```

PHP 的错误处理其实可以分为：**用户自定义错误处理** 和 **PHP标准错误处理**，两者的关系相当于两层错误捕捉器，系统会先检测是否定义了 **用户自定义错误处理**，否则会将错误交由 **PHP标准错误处理** 进行处理。

注意：PHP 的所有的 Exception 都属于 E_ERROR 级的错误，抛出时如果没有被捕获而交由 PHP 标准错误处理的话，就会 Fatal Error 导致程序退出执行。当然，PHP7 为了细化错误级别，划分了 Error 级 Error 的衍生类，这些也都属于 E_ERROR 级别的错误。

## PHP 标准错误处理

PHP 标准错误处理是在一些错误没有被用户捕获处理（没有被 try ... catch ... 或 set_error_handler 捕获处理）时，错误 会递交至 PHP 标准错误处理。相关的设置项如下：

```php
<?php
// 监听捕获的错误级别
error_reporting(E_ALL);

// 是否开启错误信息回显 将错误输出至标准输出（浏览器/命令行）
ini_set('display_errors', true);

// 死否开启错误日志记录 将错误记录至 ini：error_log 指定文件
ini_set('log_errors', true);
ini_set('error_log', __DIR__ . '/php-errors.log');
```

### 1、error_reporting([level])

获取或设定当前错误的监听级别。要注意，是获取或设定的 PHP 标准错误处理 的级别，不会有效于 try...catch... 或 set_error_handler。

### 2、display_errors

是否将错误信息回显至标准输出。默认开启，生产环境下强烈建议 **关闭** 此项。

### 3、log_errors

是否记录错误日志。默认关闭，生产环境下强烈建议 **开启** 此项。

### 4、error_log

错误日志的保存文件。注意：如果路径无效，display_errors 会被强制开启。

## PHP 用户自定义错误处理

### 1、set_error_handler

set_error_handler 并非可以捕获所有错误，且 set_error_handler 不会终止程序继续执行。处理后若返回 false，则错误会被继续递交给 PHP 标准错误处理 流程。

可以捕获： **E_WARNING & E_NOTICE & E_DEPRCATED & E_USER_\*** & 部分 **E_STRICT** 级的错误。
无法捕获： **E_ERROR & E_PARSE & E_CORE_\* & E_COMPLIE_\*** 级的错误。

有自身的错误捕获级别，默认**E_ALL | E_STRICT**，且不受 error_reporting 设定的级别的影响。这里要理解，用户自定义错误处理 和 PHP 标准错误处理 是两层错误捕捉器，有独立的捕获级别。

```php
<?php
// 用户自定义错误处理
set_error_handler(function ($error_no, $error_msg, $error_file, $error_line) {
    switch ($error_no) {
        case E_WARNING:
            $level_tips = 'PHP Warning: ';
            break;
        case E_NOTICE:
            $level_tips = 'PHP Notice: ';
            break;
        case E_DEPRECATED:
            $level_tips = 'PHP Deprecated: ';
            break;
        case E_USER_ERROR:
            $level_tips = 'User Error: ';
            break;
        case E_USER_WARNING:
            $level_tips = 'User Warning: ';
            break;
        case E_USER_NOTICE:
            $level_tips = 'User Notice: ';
            break;
        case E_USER_DEPRECATED:
            $level_tips = 'User Deprecated: ';
            break;
        case E_STRICT:
            $level_tips = 'PHP Strict: ';
            break;
        default:
            $level_tips = 'Unkonw Type Error: ';
            break;
    }

    // do some handle
    $error = $level_tips . $error_msg . ' in ' . $error_file . ' on ' . $error_line;
    echo $error . PHP_EOL;

    // 如果 return false 则错误会继续递交给 PHP 标准错误处理
    // return false;
}, E_ALL | E_STRICT);

trigger_error("用户自定义 notice error");
trigger_error("用户自定义 warning error", E_USER_WARNING);
trigger_error("用户自定义 deprecated error", E_USER_DEPRECATED);
trigger_error("用户自定义 fatal error", E_USER_ERROR);
```

### 2、trigger_error

trigger_error 用来触发用户级别的自定义错误，可以使用 set_error_handler 捕获处理。
默认的错误级别是 **E_USER_NOTICE**，我们可以自定义。
这里需要注意的是：**E_USER_ERROR** 级别的错误如果被 PHP 标准错误处理 捕获，脚本也会退出执行错误。

```php
<?php
error_reporting(E_ALL);
ini_set('display_errors', true);

trigger_error('用户自定义 notice error', E_USER_NOTICE);
echo 'continue A' . PHP_EOL;
trigger_error('用户自定义 warning error', E_USER_WARNING);
echo 'continue B' . PHP_EOL;
trigger_error('用户自定义 deprecated error', E_USER_DEPRECATED);
echo 'continue C' . PHP_EOL;
trigger_error('用户自定义 fatal error', E_USER_ERROR);
echo 'D point will not be executed' . PHP_EOL;
```

### 3、set_exception_handler

set_exception_handler 用户自定义捕获异常 handler，异常没有被 try ... catch 捕获处理的话会被抛出，此时系统会检查上下文是否注册了 set_exception_handler。
如果未注册 则进入 PHP 标准错误处理 致命错误退出执行。
如果已注册 则进入 set_exception_handler 处理 程序依然会退出执行。
而 try ... catch ... 捕获异常后仍不会退出执行，故强烈建议将有异常的执行逻辑放入 try ... catch 中

```php
<?php
// 捕获异常后程序会退出执行
set_exception_handler(function ($exception) {
    echo $exception;
    // 此处程序会退出执行 异常到此结束 并不会交给 PHP 标准异常处理
});

throw new Exception('hello world!');

echo 'will i be executed？';
```

### 4、try ... catch ...

开发中用户层面的 set_error_hanlder 无法捕获的错误还剩下 **E_ERROR** 和 **E_PARSE** 两个级别，使用 try ... catch ... 则可以将这俩货捕捉到。

> 来个小插曲：大家对 E_PARSE 熟悉又陌生，可能经常遇到（各大框架都有此级别错误捕获提示），但自己不知道如何捕获，其实首先要理解 E_PARSE 错误的发生时段。
>
> E_PARSE：即语法解析错误，Syntax Error then Parse Error，PHP 将脚本载入 Zend Engine 后，最开始要做的就是检查基本语法是否有误，无误才会调用解释器，一行行的开始解释运行。
>
> 这里就有个鸡生蛋，蛋生鸡的问题了。如下代码：
>
> ```
> <?php
> //认为关闭了错误捕获 不应该有错误报出才对
> error_reporting(0);
>
> echo 'i lose semicolon'
> ```
>
> 然后很多人会诧异，明明关闭了错误提示，为什么还会报错？
>
> 没错，代码的确正确的书写了关闭错误提示的意图。但还没被执行到时，脚本就因为最初始的语法解析错误，被 Zend Engine 抛出 Parse Error 终止执行了。同时还要理解，PHP include / require 只有在真的解释到这一行代码时，引用的文件才会被载入--解析--解释执行。
>
> 所以，我们需要有一个无错的 try ... catch ... 容器，在容器中便可以对后续引用的外部脚本进行 E_PARSE 错误捕捉。
>
> 例如框架自身是一个无错的运行容器，开发者自写的 MVC 是被 include / require 到此容器中 解析 -- 解释执行 的，用户代码的语法错误即会被容器的 try ... catch ... 优雅的捕获到。

try ... catch ... 的错误捕获级别同样不受 error_reporting 影响，我们可以通过 多层 catch 细化各类型的错误。

```php
<?php

// typeError demo
function foo(): int
{
    return 'hello world';
}

try {
    //foo();
    // echo strlen('hello world', 233);
} catch (\ErrorException $errorException) {
    // 捕获错误异常
    echo 'ErrorException: ' . $errorException . PHP_EOL;
} catch (\Exception $exception) {
    // 捕获异常
    echo 'Exception: ' . $exception . PHP_EOL;
} catch (\TypeError $typeError) {
    // 捕获类型错误 返回值/参数类型不正确 declare(strict_types = 1) 严格模式下更容易出现
    echo 'Type Error: ' . $typeError . PHP_EOL;
} catch (\ParseError $parseError) {
    // 捕获解析错误 语法错误
    echo 'Parse Error: ' . $parseError . PHP_EOL;
} catch (\DivisionByZeroError $divisionByZeroError) {
    // 除 0 无法捕获 但 除 0 取余可以捕获 = = 很无奈
    echo 'Division By Zero Error: ' . $divisionByZeroError . PHP_EOL;
} catch (\Error $error) {
    // 基本错误
    echo 'Error: ' . $error . PHP_EOL;
}
```

这里要注意的是，DivisionByZeroError 在 PHP7 中依然无法隐式的完美捕获。准确的说：

> 当 x / 0 时抛出一个 E_WARNING 级别的错误，我们可以 set_error_handler 捕获，然后再判断错误为
>
> 'Devision by zero' 时抛出一个 ErrorException 的异常交由 try ... catch ... 捕获处理即可。
>
> 当 x % 0 时才会直接抛出 DivisionByZeroError 的错误。
> 或者使用 intdiv(x, 0) 来代替 x / 0，会自动抛出 DivisionByZeroError

当然，你也可以显示的判断除数为 0 来决定是否抛出个 DivisionByZeroError

```php
<?php
try {
    $divisor = 0;
    if ($divisor == 0) {
        throw new DivisionByZeroError('Division by zero！');
    }
    echo 233 / $divisor;
} catch (\DivisionByZeroError $error) {
    echo $error;
}
```

## PHP 预定义的 Error 和 Exception

Predefined Exceptions 预定义异常 可由系统自动抛出
[http://php.net/manual/en/rese...](http://php.net/manual/en/reserved.exceptions.php)

> Exception
> ErrorException
> Error
> ArgumentCountError
> ArithmeticError
> AssertionError
> DivisionByZeroError
> ParseError
> TypeError

SPL Exceptions SPL 标准规范异常 可供开发者规范代码自行抛出
[http://php.net/manual/en/spl....](http://php.net/manual/en/spl.exceptions.php)

> BadFunctionCallException
> BadMethodCallException
> DomainException
> InvalidArgumentException
> LengthException
> LogicException
> OutOfBoundsException
> OutOfRangeException
> OverflowException
> RangeException
> RuntimeException
> UnderflowException
> UnexpectedValueException

## 完善的错误和异常捕获

下面的代码基本呈现和捕获了 PHP7 提供的所有预定义错误和异常 及 PHP 标准错误处理
1、使用 try ... catch 捕获 E_ERROR 及 E_PARSE 级别的 Error (及Error 类的衍生类) 和 Exception (ErrorException)。
2、对于 try ... catch 无法捕获的 E_WARNING，E_NOTICE，E_DEPRECATED，E_USER_*，部分 E_STRICTED 级别的错误，我们使用 set_error_handler 捕获处理，捕获后我们其实可以将错误信息封装到 ErrorException 中并抛出，这样处理流又会交给 try ... catch，可以统一处理，比如Yii2框架就是这样处理的。
3、set_exception_handler 则是捕获在没有 try ... catch 中执行的代码的异常，所以强烈建议一些存在异常 风险的逻辑要放入 try ... catch 中。

```php
<?php
// php >= 7
// PHP 标准错误处理 捕获级别
error_reporting(E_ALL);
// 是否将 标准错误处理 捕获的错误回显在 stdout 上
ini_set('display_errors', false);
// 开启错误日志
ini_set('log_errors', true);
// 如果错误日志路径无效 display_errors 依然会强制打开
ini_set('error_log', __DIR__ . '/php-errors.log');

/**
 * set_error_handler 用户自定义错误 handler
 * 能够捕获 E_WARNING E_NOTICE E_DEPRECATED E_USER_* E_STRICT 级的错误
 * 无法捕获 E_ERROR E_PARSE E_CORE_* E_COMPILE_* [DivisionByZeroError TypeError] 级的错误
 */
set_error_handler(function ($error_no, $error_msg, $error_file, $error_line) {
    switch ($error_no) {
        case E_WARNING:
            // x / 0 错误 PHP7 依然不能很友好的自动捕获 只会产生 E_WARNING 级的错误
            // 捕获判断后 throw new DivisionByZeroError($error_msg)
            // 或者使用 intdiv(x, 0) 方法 会自动抛出 DivisionByZeroError 的错误
            if (strcmp('Division by zero', $error_msg) == 0) {
                throw new \DivisionByZeroError($error_msg);
            }

            $level_tips = 'PHP Warning: ';
            break;
        case E_NOTICE:
            $level_tips = 'PHP Notice: ';
            break;
        case E_DEPRECATED:
            $level_tips = 'PHP Deprecated: ';
            break;
        case E_USER_ERROR:
            $level_tips = 'User Error: ';
            break;
        case E_USER_WARNING:
            $level_tips = 'User Warning: ';
            break;
        case E_USER_NOTICE:
            $level_tips = 'User Notice: ';
            break;
        case E_USER_DEPRECATED:
            $level_tips = 'User Deprecated: ';
            break;
        case E_STRICT:
            $level_tips = 'PHP Strict: ';
            break;
        default:
            $level_tips = 'Unkonw Type Error: ';
            break;
    }

    // do some handle
    $error = $level_tips . $error_msg . ' in ' . $error_file . ' on ' . $error_line;
    echo $error . PHP_EOL;

    // or throw a ErrorException back to try ... catch block
    // throw new \ErrorException($error);

    // 如果 return false 则错误会继续递交给 PHP 标准错误处理
    // return false;
}, E_ALL | E_STRICT);

/**
 * set_exception_handler 用户自定义捕获异常 handler
 * 异常没有被 try ... catch ... 捕获处理的话会被抛出
 * 此时系统会检查上下文是否注册了 set_exception_handler
 * 如果未注册 则进入 PHP 标准异常处理 致命错误退出执行
 * 如果已注册 则进入 set_exception_handler 处理 程序依然会退出执行
 * 而 try ... catch ... 捕获异常后仍不会退出执行
 * 故强烈建议将有异常的执行逻辑放入 try ... catch 中
 */
set_exception_handler(function ($exception) {
    echo $exception;
    // 此处程序会退出执行 异常到此结束 并不会交给 PHP 标准异常处理
});

// type error demo
function foo(int $bar): int
{
    return 'result type error';
}

// 捕获 E_ERROR E_PARSE 级的 Error
// 捕获 Exception
try {
    // 加载外部文件的正确写法
    $file = __DIR__ . '/bar.inc.php';
    if (file_exists($file)) {
        require_once $file;
    } else {
        throw new Exception($file . ' not exists!');
    }

    // ParseError 解析错误
    // bar.inc.php 的内容要有基本的语法错误: <?php echo 'some syntax error'

    // ArgumentCountError extends TypeError PHP >= 7.1.0
    // strlen 参数错误
    echo strlen('hello world', 4);

    // TypeError 类型错误
    // foo 要求的参数为整形，传递了字符串
    // 返回类型为 int 但 return 了 string 类型错误
    foo("type error");

    // DivisionByZeroError extends ArithmeticError
    // x / 0 会抛出 E_WARNING 的异常 但不会自动抛出 DivisionByZeroError
    // 我们可以使用 set_error_handler 进行捕获然后手动抛出 DivisionByZeroError
    1 / 0;
    // Integer Divison 等同于 1 / 0 可以直接抛出 DivisionByZeroError
    intdiv(1, 0);
    // 除 0 取余 可以直接抛出 DivisionByZeroError
    1 % 0;

    // ArithmeticError 错误
    intdiv(PHP_INT_MIN, -1);

    // AssertionError 断言错误
    assert('1 != 1');

    // 调用未定义的函数 错误级别：E_ERROR
    bar();
} catch (\ErrorException $errorException) {
    // 错误异常
    // 最常用的就是将那几个非致命的错误捕获后 ErrorException 回抛到 try ... catch 中
    echo 'ErrorException: ' . $errorException . PHP_EOL;
} catch (\Exception $exception) {
    // 基本异常
    echo 'Exception: ' . $exception . PHP_EOL;
} catch (\ParseError $parseError) {
    // 解析错误 语法错误
    echo 'Parse Error: ' . $parseError . PHP_EOL;
} catch (\ArgumentCountError $argumentCountError ) {
    // 传参非法错误 php >= 7.1.0
    echo 'Argument Count Error: ' . $argumentCountError . PHP_EOL;
} catch (\TypeError $typeError) {
    // 类型错误 返回值
    echo 'Type Error: ' . $typeError . PHP_EOL;
} catch (\DivisionByZeroError $divisionByZeroError) {
    // x / 0 不抛出  x % 0 可以抛出
    // x / 0 可以用 intdiv(x, 0) 代替 会抛出
    echo 'Division By Zero Error: ' . $divisionByZeroError . PHP_EOL;
} catch (\ArithmeticError $arithmeticError) {
    // 算数运算错误 intdiv(PHP_INT_MIN, -1) 触发
    echo 'Arithmetic Error: ' . $arithmeticError . PHP_EOL;
} catch (\AssertionError $assertionError) {
    // 断言错误
    echo 'Assertion Error: ' . $assertionError . PHP_EOL;
} catch (\Error $error) {
    // 基本错误
    echo 'Error: ' . $error . PHP_EOL;
}

echo "run finished!" . PHP_EOL;
```

## 总结

1、PHP 允许用户自定义 Error 和 Exception 的捕获与处理。如用户未捕获处理，则会递交给 PHP 标准错/异常处理，根据 errror_reporting display_errors log_erros error_log 参数决定处理方式。生产环境应关闭 display_errors 同时开启 log_errors 记录错误日志。

2、set_error_handler 可以捕获 E_WARNING & E_NOTICE & E_DEPRECATED & E_USER_* 和 部分 E_STRICT 级的错误。set_error_handler 如果返回了 false 错误会递交给 PHP 标准错误处理。set_error_handler 不会终止程序执行。

3、trigger_error 可以用来抛出用户级的错误，且 E_USER_ERROR 效用等同于 E_ERROR，PHP 标准错误处理 捕获此级别的错误时会终止程序执行。

4、set_exception_handler 用户自定义异常捕获，捕获后程序依然会终止运行，但不会再将异常递交给 PHP 标准异常处理。

5、try ... catch 可以捕获所有的 Exception 和 E_ERROR & E_PARSE 级的错误。程序不会退出执行。

6、PHP 自带了一些 Predefined Exceptions，同时有规范一些 SPL Exceptions，供开发者规范自己的错误异常架构。



转载自 https://segmentfault.com/a/1190000014977430

