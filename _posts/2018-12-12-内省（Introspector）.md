---
title: 内省（Introspector）
date: 2018-12-12
author: 丶德灬锅
tags: 内省

---

# 内省（Introspector）

[TOC]

> **内省（Introspector）**是Java语言对JavaBean类属性、事件的一种缺省处理方法。
>
> **JavaBean**是一种特殊的类，主要用于传递数据信息，这种类中的方法主要用于访问私有的字段，且方法名符合某种命名规则。如果要在两个模板之间传递多个信息，可将这些信息封装到一个JavaBean中，这种JavaBean的实例对象通常称之为“值对象"（Value Object，简称VO），这些信息在类中用私有字段来储存，如果读取或设置这些字段的值，则需要通过相应的Getter/Setter方法来访问。
>
> 例如：类A中有属性name，那我们可以通过getName/setName来得到其值或者设置其值。 通过getName/setName来访问name属性，这就是默认的规则。Java JDK提供了一套API来访问某个属性的 Getter/Setter方法，这就是内省。

## 内省和反射的区别

内省机制是通过反射实现的。

## JDK API[^1]

位于java.bean包中

### PropertyDescriptor

```java
PropertyDescriptor propertyDescriptor = new PropertyDescriptor(propertyName, UserInfo.class);
Method writeMethod = propertyDescriptor.getWriteMethod();
// ...
PropertyDescriptor propertyDescriptor = new PropertyDescriptor(propertyName, UserInfo.class);
Method readMethod = propertyDescriptor.getReadMethod();
```

### Introspector

```java
BeanInfo beanInfo = Introspector.getBeanInfo(UserInfo.class);
PropertyDescriptor[] propertyDescriptors = beanInfo.getPropertyDescriptors();
// ...遍历
```

## BeanUtils[^1]

内省操作非常繁琐，Apache开发了一套简单、易用的API来操作Bean的属性，BeanUtils工具包。

```java
UserInfo userInfo = new UserInfo();
BeanUtils.setProperty(userInfo, "userId", userId);
BeanUtils.setProperty(userInfo, "userName", userName);
BeanUtils.setProperty(userInfo, "age", age);
```

## Getter/Setter方法 vs 内省

在Servelet中通过request.getParameter()给对象赋值时，需要使用Setter方法一一赋值比较繁琐，而使用内省可以抽象为更加通用方法。

[^1]: [阿里云Code代码](https://code.aliyun.com/lideyu/j2se/tree/master/basic/src/main/java/com/ldy/advanced/introspector)
