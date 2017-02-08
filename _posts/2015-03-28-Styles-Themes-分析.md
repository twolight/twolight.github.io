---
layout:     post
title:      Looper Handler MessageQueue源码分析
date:       2015-03-28 23:10:00
summary:    Style和Theme的定义和使用以及相关知识
categories: jekyll pixyll
---

## Styles and Themes

* ### Style和Theme的定义和使用  


>A style is a collection of properties that specify the look and format for a View or window. A style can specify properties such as height, padding, font color, font size, background color, and much more. A style is defined in an XML resource that is separate from the XML that specifies the layout.

Style主要应用于`View`和`Window`

>A theme is a style applied to an entire Activity or application, rather than an individual View (as in the example above). When a style is applied as a theme, every View in the Activity or application will apply each style property that it supports. 


Theme主要应用于`Activity`和`Application`，当一个style作用在theme中，Activity或Application中的每个View都将具有这个style的所有属性。

</br>**Theme实质上是Style，只是作用场景不同**  </br></br>



举个例子

```
	<TextView
	    android:layout_width="fill_parent"
	    android:layout_height="wrap_content"
	    android:textColor="#00FF00"
	    android:typeface="monospace"
	    android:text="@string/hello" />
 ```
将需要重用的属性值抽出到Style文件中

```
	<?xml version="1.0" encoding="utf-8"?>
	<resources>
	    <style name="CodeFont" parent="@android:styleTextAppearance.Medium">
	        <item name="android:layout_width">fill_parent</item>
	        <item name="android:layout_height">wrap_content</item>
	        <item name="android:textColor">#00FF00</item>
	        <item name="android:typeface">monospace</item>
	    </style>
	</resources>
```
使用Style之后

```
	<TextView
    	style="@style/CodeFont"
    	android:text="@string/hello" />
```
</br>
> The parent attribute in the \<style\> element is optional and specifies the resource ID of another style from which this style should inherit properties. You can then override the inherited style properties if you want to.  

Style也可以像面向对象一样继承，父style是可选的，子style可以覆写父style的属性。</br>  
Android官方的Style，通过parent方式来继承的，如果是自己定义的style，可以通过下面的方式继承  

```
	<style name="CodeFont.Red">
       <item name="android:textColor">#FF0000</item>
	</style>
```


**实际的开发中，应用中相同的按钮，文字，控件都可以采用相同的Style。**</br></br>




* ### Theme兼容性

Android不同的版本下有不同的Theme，如何兼容呢。

res/values/styles.xml文件中定义如下Theme

```
	<style name="LightThemeSelector" parent="android:Theme.Light">
    	...
	</style>
```
当我们想在Android3.0（API 11）以上使用新的Theme则可以res/values-v11目录下定义如下Theme

```
	<style name="LightThemeSelector" parent="android:Theme.Holo.Light">
    	...
	</style>

```

这样当我们编译的APK在不同的设备上运行时就能自己切换选择适合自己平台的Theme了。



* ### Style和Theme属性
</br>

	
	>To apply a style definition as a theme, you must apply the style to an Activity or application in the Android manifest. When you do so, every View within the Activity or application will apply each property that it supports. For example, if you apply the CodeFont style from the previous examples to an Activity, then all View elements that support the text style properties will apply them. Any View that does not support the properties will ignore them. If a View supports only some of the properties, then it will apply only those properties.
	
	不同的View支持的属性不同，View的某些属性，Activity和Window并不支持，如果有不支持的属性会被忽略，哪些属性支持还是不支持，这就需要我们去自己去查询API。</br></br>
	
	* Theme 查阅API的R.styleable进行选择
	* Style 查阅API的R.attr进行选择
	
	
	
 



