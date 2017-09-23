---
layout: post
title: spl_autoload_register 函数详解
description: spl_autoload_register — 注册__autoload()函数 
---

> spl_autoload_register
> (PHP 5 >= 5.1.2)

> spl_autoload_register — 注册__autoload()函数

### 说明  
> bool spl_autoload_register ([ callback $autoload_function ] )  
将函数注册到SPL __autoload函数栈中。如果该栈中的函数尚未激活，则激活它们。  

如果在你的程序中已经实现了__autoload函数，它必须显式注册到__autoload栈中。因为spl_autoload_register()函数会将Zend Engine中的__autoload函数取代为spl_autoload() 或 spl_autoload_call()。  

### 参数  
> autoload_function  
欲注册的自动装载函数。如果没有提供任何参数，则自动注册autoload的默认实现函数spl_autoload()。  

### 返回值  
> 如果成功则返回 TRUE，失败则返回 FALSE。  

注：SPL是Standard PHP Library(标准PHP库)的缩写。它是PHP5引入的一个扩展库，其主要功能包括autoload机制的实现及包括各种Iterator接口或类。SPL autoload机制的实现是通过将函数指针autoload_func指向自己实现的具有自动装载功能的函数来实现的。SPL有两个不同的函数spl_autoload, spl_autoload_call，通过将autoload_func指向这两个不同的函数地址来实现不同的自动加载机制。  

```language-php
    classLOAD  
    {  
        staticfunctionloadClass($class_name)  
        {  
            $filename= $class_name.".class.php";  
     		$path= "include/".$filename  
            if(is_file($path)) returninclude$path;  
        }  
    }  
       
    /** 
     * 设置对象的自动载入 
     * spl_autoload_register — Register given function as __autoload() implementation 
     */  
    spl_autoload_register(array('LOAD', 'loadClass'));  
       
    /** 
    *__autoload 方法在 spl_autoload_register 后会失效，因为 autoload_func 函数指针已指向 spl_autoload 方法 
    * 可以通过下面的方法来把 _autoload 方法加入 autoload_functions list 
    */  
    spl_autoload_register( '__autoload');  

```

如果同时用spl_autoload_register注册了一个类的方法和__autoload函数，那么会根据注册的先后，如果在第一个注册的方法或函数里加载了类文件，就不会再执行第二个被注册的类的方法或函数。反之就会执行第二个被注册的类的方法或函数。  

```language-php
    <?php    
    class autoloader {    
        public static $loader;    
            
        public static function init() {    
            if (self::$loader == NULL)    
                self::$loader = new self ();    
                
            return self::$loader;    
        }    
            
        public function __construct() {    
            spl_autoload_register ( array ($this, 'model' ) );    
            spl_autoload_register ( array ($this, 'helper' ) );    
            spl_autoload_register ( array ($this, 'controller' ) );    
            spl_autoload_register ( array ($this, 'library' ) );    
        }    
            
        public function library($class) {    
            set_include_path ( get_include_path () . PATH_SEPARATOR . '/lib/' );    
            spl_autoload_extensions ( '.library.php' );    
            spl_autoload ( $class );    
        }    
            
        public function controller($class) {    
            $class = preg_replace ( '/_controller$/ui', '', $class );    
                
            set_include_path ( get_include_path () . PATH_SEPARATOR . '/controller/' );    
            spl_autoload_extensions ( '.controller.php' );    
            spl_autoload ( $class );    
        }    
            
        public function model($class) {    
            $class = preg_replace ( '/_model$/ui', '', $class );    
                
            set_include_path ( get_include_path () . PATH_SEPARATOR . '/model/' );    
            spl_autoload_extensions ( '.model.php' );    
            spl_autoload ( $class );    
        }    
            
        public function helper($class) {    
            $class = preg_replace ( '/_helper$/ui', '', $class );    
                
            set_include_path ( get_include_path () . PATH_SEPARATOR . '/helper/' );    
            spl_autoload_extensions ( '.helper.php' );    
            spl_autoload ( $class );    
        }    
       
    }    
       
    //call    
    autoloader::init ();    
    ?>    

```