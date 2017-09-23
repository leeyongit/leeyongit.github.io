---
layout: post
title: Laravel5 笔记
description: Laravel是一套简洁、优雅的PHP Web开发框架(PHP Web Framework)。它可以让你从面条一样杂乱的代码中解脱出来；它可以帮你构建一个完美的网络APP，而且每行代码都可以简洁、富于表达力。
---


## 安装Laravel  
```language-php
composer create-project laravel/laravel blog
```

## 创建控制器  
```language-php
php artisan make:controller ArticleController
```

## 向视图中传递变量
### 使用with()方法
```language-php
<?php
	puclic function index() {
		$title = '标题';
		return view('articles.lists')->with('title', $title);
	}
```

模板输出变量 {{ $title }}
{{ }} 符号会将数据原样输出
页面元素渲染输出 {!! $title !!}

### 直接给view()传参数
```language-php
<?php
	public function index()
	{
		$title = '<span style="color: red">文章</span>标题1';
		return view('articles.lists',['title'=>$title]);
	}
```

### 使用compact  
```language-php
<?php
    public function index()
    {
        $title = '<span style="color: red">文章</span>标题1';
        $intro = '文章一的简介';
        return view('articles.lists',compact('title','intro'));
    }
```
compact() 的字符串可以就是变量的名字，多个变量名用逗号隔开

## Blade的基本用法  
```language-php
@yield()
@extends()
@if() and @unless()
@foreach()
```


## 创建一个migration文件  
```language-php
php artisan make:migration create_articles_table --create='articles'
php artisan migrate
php artisan migrate:rollback
php artisan make:model Article
```