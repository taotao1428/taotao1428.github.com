---
layout:     post
title:      "对windphp-9.0.1的代码解析"
subtitle:   "前端"
date:       2016-05-08
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 生活
---


#对windphp-9.0.1的代码解析
##今天我看的文件是global.js

首先他定义了公共方法Wind.Util

1.ajaxBtnDisable作用：在点击会出发ajax的按钮时，调用该函数可以使按钮处于`disable`状态（即不可以点击后不会触发事件）

* 在第一个按钮添`中...`，提醒用户已经在提交表单
* 给btn添加属性`disabled=ture`，使其不能在触发事件
* 给btn添加样式类`disabled`，
* 添加数据`sublock=true`，暂时还不知道这个的用处

```javascript
    ajaxBtnDisable : function(btn){
    	    //按钮提交不可用
		var textnode = document.createTextNode('中...');
		btn[0].appendChild(textnode);
		btn.prop('disabled', true).addClass('disabled').data('sublock', true);
	},
```


