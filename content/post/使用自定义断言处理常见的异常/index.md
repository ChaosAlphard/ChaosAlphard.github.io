---
title: 使用自定义断言处理常见的异常
date: 2020-10-19 23:37:23
tags: [Java]
categories: [编程,Java]
image: title.jpg
---

　　`Java`中空判断随处可见，通常我们都是手动判断是否为空，然后再抛出自定义异常以及记录信息，例如:
```java
var data = dao.findById(id);
if(data == null) {
  log.error("数据为空, id{}", id);
  throw new CustomException(Status.USER_NOT_EXIST);
}
```
这样的代码写多了就觉得麻烦，可不可以简化一下呢

# 使用Assert替代null判断
　　我们都知道sprng中有一个`Assert`类，其中有一个方法`notNull()` 可以用来判断是否为空，并且输出自定义提示，何不根据此改造一下，做一个自定义的`Assert`类

## 创建枚举类接口
创建默认枚举方法，需要与你自定义枚举中的属性对应
```java
public interface IBaseEnum {
    int getCode();

    String getMessage();
}
```

<!-- more -->

## 创建自定义异常
```java
public class CustomException extends Exception {
    private int code;
    private String detail;
    private Object data;

    public CustomException(IBaseEnum enums, String detail, Object data) {
        super(enums.getMessage());
        this.code = enums.getCode();
        this.detail = detail;
        this.data = data;
    }

    public CustomException(int code, String message, String detail, Object data) {
        super(message);
        this.code = code;
        this.detail = detail;
        this.data = data;
    }

    @Override
    public String toString() {
        return "CustomException{" + "code=" + code + ", message='" + super.getMessage() + '\'' + ", detail='" + detail + '\'' + ", data=" + data + "} ";
    }

    public int getCode() {
        return code;
    }

    public String getDetail() {
        return detail;
    }

    public Object getData() {
        return data;
    }
}
```

## 创建异常断言构造
创建默认异常的构造方法，可以根据情况修改
```java
public interface ICustomExceptionAssert {

    default CustomException newException(IBaseEnum enums, String detail, Object data) {
        return new CustomException(enums, detail, data);
    }

    default CustomException newException(int code, String message, String detail, Object data) {
        return new CustomException(code, message, detail, data);
    }
}
```

## 创建基础断言类
默认的异常断言，可以根据自己业务需要增加各种逻辑判断
```java
public interface IBaseAssert extends IBaseEnum, ICustomExceptionAssert {

    default void notNull(Object object, String detail) throws CustomException {
        if(object == null) {
            LogUtils.error(detail)
            throw newException(this, detail, null);
        }
    }

    default void success(int line, String detail) throws CustomException {
        if(line < 1) {
            LogUtils.error(detail)
            throw newException(this, detail, data);
        }
    }
}
```

## 使用枚举来完善断言
```java
public enum GlobalAssert implements IBaseAssert {
    USER_EXIST(Status.USER_NOT_EXIST,"用户不存在"),
    SQL_EXECUTE(Status.SQL_ERROR, "无匹配记录"),
    ;

    private int code;
    private String message;

    Assert(int code, String message) {
        this.code = code;
        this.message = message;
    }

    @Override
    public int getCode() {
        return this.code;
    }

    @Override
    public String getMessage() {
        return this.message;
    }
}
```

## 测试
```java
var data = dao.findById(id);
var logInfo = MessageFormatter.arrayFormat("数据为空, id: {}", id).getMessage();
Assert.USER_EXIST.notNull(data, logInfo);
var sqlExecErr = MessageFormatter.arrayFormat("新增用户失败: {}", data).getMessage();
Assert.SQL_EXECUTE.success(other.insert(data), sqlExecErr)
```

![结果](01.jpg)

## 异常处理

使用`@ControllerAdvice`与`@ExceptionHandler`注解来进行全局异常捕获即可
