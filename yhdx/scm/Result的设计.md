1、为什么设计Result这个类？

（1）、业务层xxxService的返回值，如果返回值是domain领域对象，则一些特殊或异常情况下，如果不通过抛出异常的方式，或者在domain上增加额外侵入式的代码，返回结果携带的信息无法涵盖所有的情况告诉调用者。首先异常不要用来做流程控制，条件控制。异常设计的初衷是解决程序运行中的各种意外情况，且异常的处理效率比条件判断方式要低很多（见《阿里巴巴Java开发手册》）。 其次在domain对象上增加额外的字段记录返回情况（包括异常），并没有提供一套统一的解决方案。

（2）、对外接口层的返回值，如果代码运行发生了异常，直接抛给外部调用方，并不友好且容易让调用方感到迷惑不知如何处理。

（3）、返回值如果是Result类型，可以基于方法返回类型做一些系统级别的统一处理，例如aop，统一分析处理返回值Result。



2、Result类

```java
public class Result<T> {
    /**
     * 编码，正常返回ok；否则返回具体的错误编码，方便调用方进行判断处理。
     */
	private String code;
    /**
     * 提示信息，用于code字面上的解释，也可以用于返回客户端进行结果提示。
     */
    private String msg;
    /**
     * 一般正常情况下，返回的真实数据结果。
     */
    private T result;
    ......
}
```



3、代码示例

```java
public Result<User> createUser(String userCode, String password) {
    if(!StringUtils.hasText(userCode)) {
        return Result.errorWithMsg("bad-request", "用户编码不能为空");
    }
    if(userService.isExist(userCode)) {
        return Result.errorWithMsg("conflict", "用户编码已存在");
    }
    User user = new User();
    user.setUserCode(userCode);
    ......
    userService.save(user);
    return Result.result(user);
}
```

