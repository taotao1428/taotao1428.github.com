---
layout:     post
title:      "对页面元素拖动的实现"
subtitle:   "前端"
date:       2016-06-09
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 学习
---


#页面元素拖动的实现

##通过写jq扩展函数的方式实现

扩展一个名为drag的扩展函数

1.定义变量,并对变量进行初始化

```javascript
var startDrag = false, //是否开始拖动
	startMX, startMY, //开始拖动时鼠标的位置
	startEX, startEY, //开始拖动时元素的位置
	parentW, parentH, //父级元素的大小
	eleW, eleH, //元素的大小
	defaults = {overflow: false}, //是否可以拖到父级元素外
	$self = $(this); 

options = $.type(options) == 'object' ? $.extend(defaults, options) : defaults;

parentW = $self.offsetParent().width();
parentH = $self.offsetParent().height();
eleW = $self.innerWidth();
eleH = $self.innerHeight();
```

2.监听mousedown事件，将`startDrag`设置为`true`

```javascript
$self.mousedown(function(e){
	e.stopPropagation(); //阻止父级可拖动元素触发事件
	startDrag = true;
	startMX   = e.pageX;
	startMY   = e.pageY;
	startEX   = $(this).position().left;
	startEY   = $(this).position().top;

})
```

3.监听mouseup事件，将`startDrag`设置为`false`

```javascript
	$(document).mouseup(function(){
		startDrag = false;
	})
```

4.监听mousemove事件，执行函数

```javascript
$(document).mousemove(function(e){
	if(!startDrag) return; //如果鼠标没按下，不执行函数

	var top, left;
	DX   = e.pageX - startMX;
	DY   = e.pageY - startMY;
	if(!options.overflow){
		top  = startEY + DY;
		top  = top<0 ? 0 : ((top+eleH)>parentH ? (parentH-eleH) : top);
		left = startEX + DX;
		left = left<0 ? 0 : ((left+eleW)>parentW ? (parentW-eleW) : left);
	}

	$self.css({top:top+"px", left:left+"px"});
})
```

##全部代码

```javascript
$.fn.extend({
	drag: function(options){
		var startDrag = false, //是否开始拖动
			startMX, startMY, //开始拖动时鼠标的位置
			startEX, startEY, //开始拖动时元素的位置
			parentW, parentH, //父级元素的大小
			eleW, eleH, //元素的大小
			defaults = {overflow: false}, //是否可以拖到父级元素外
			$self = $(this); 

		options = $.type(options) == 'object' ? $.extend(defaults, options) : defaults;

		parentW = $self.offsetParent().width();
		parentH = $self.offsetParent().height();
		eleW = $self.innerWidth();
		eleH = $self.innerHeight();

		$self.mousedown(function(e){
			e.stopPropagation();
			startDrag = true;
			startMX   = e.pageX;
			startMY   = e.pageY;
			startEX   = $(this).position().left;
			startEY   = $(this).position().top;

		})

		$(document).mouseup(function(){
			startDrag = false;
		})

		$(document).mousemove(function(e){
			if(!startDrag) return; //如果鼠标没按下，不执行函数

			var top, left;
			DX   = e.pageX - startMX;
			DY   = e.pageY - startMY;
			if(!options.overflow){
				top  = startEY + DY;
				top  = top<0 ? 0 : ((top+eleH)>parentH ? (parentH-eleH) : top);
				left = startEX + DX;
				left = left<0 ? 0 : ((left+eleW)>parentW ? (parentW-eleW) : left);
			}

			$self.css({top:top+"px", left:left+"px"});
		})
	}
})
```


