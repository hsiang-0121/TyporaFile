## springboot 统一返回类

***

### 定义返回标准格式

1. status 状态值：由后端统一定义各种返回结果的状态码
2. message 描述：本次接口调用的结果描述
3. data 数据：本次返回的数据。

```json
{
  "status":"100",
  "message":"操作成功",
  "data":"hello"
}
```



### 定义返回对象

~~~java
@Data
public class DataResult<T> {
    /**
     * 响应状态码
     */
    @ApiModelProperty(value = "响应状态码")
    private int code;

    /**
     * 响应提示语
     */
    @ApiModelProperty(value = "响应提示语")
    private String msg;

    /**
     * 响应数据
     */
    @ApiModelProperty(value = "响应数据")
    private T data;

    public DataResult(int code, String msg, T data) {
        this.code = code;
        this.msg = msg;
        this.data = data;
    }

    public DataResult(int code, T data) {
        this.code = code;
        this.data = data;
        this.msg=null;
    }

    public DataResult(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public DataResult() {
        this.code= BaseResponseCode.SUCCESS.getCode();
        this.msg=BaseResponseCode.SUCCESS.getMsg();
        this.data=null;
    }

    public DataResult(T data) {
        this.data = data;
        this.code=BaseResponseCode.SUCCESS.getCode();
        this.msg=BaseResponseCode.SUCCESS.getMsg();
    }

    public DataResult(ResponseCodeInterface responseCodeInterface) {
        this.data = null;
        this.code = responseCodeInterface.getCode();
        this.msg = responseCodeInterface.getMsg();
    }

    public DataResult(ResponseCodeInterface responseCodeInterface, T data) {
        this.data = data;
        this.code = responseCodeInterface.getCode();
        this.msg = responseCodeInterface.getMsg();
    }
    
    /**
     * 操作成功 data为null
     * @throws
     */
    public static DataResult success(){
        return new DataResult();
    }
    
    /**
     * 操作成功 data 不为null
     * @throws
     */
    public static <T>DataResult success(T data){
        return new DataResult(data);
    }

    public static <T>DataResult error(int code, String msg, T data) {
        return  new DataResult(code, msg, data);
    }
    
    /**
     * 自定义 返回操作 data 可控
     * @throws
     */
    public static <T>DataResult getResult(int code,String msg,T data){
        return new DataResult(code,msg,data);
    }
    
    /**
     *  自定义返回  data为null
     * @throws
     */
    public static DataResult getResult(int code,String msg){
        return new DataResult(code,msg);
    }
    
    /**
     * 自定义返回 入参一般是异常code枚举 data为空
     * @throws
     */
    public static DataResult getResult(BaseResponseCode responseCode){
        return new DataResult(responseCode);
    }
    
    /**
     * 自定义返回 入参一般是异常code枚举 data 可控
     * @throws
     */
    public static <T>DataResult getResult(BaseResponseCode responseCode, T data){

        return new DataResult(responseCode,data);
    }

}
~~~



### 定义状态码

~~~java
public enum BaseResponseCode implements ResponseCodeInterface {
    /**
     * 这个要和前端约定好
     */
    SUCCESS(0,"操作成功!"),
    SYSTEM_ERROR(50001,"系统异常请稍后再试!"),
	;

    BaseResponseCode(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    /**
     * 状态码
     */
    private final int code;
    /**
     * 响应提示
     */
    private final String msg;

    @Override
    public int getCode() {
        return code;
    }

    @Override
    public String getMsg() {
        return msg;
    }
}
~~~



### 统一返回格式

~~~java
	@GetMapping("test")
    private DataResult<> test() {
        return DataResult.success("hello");
    }
~~~

在controller层需要对每个接口进行`DataResult.success()` 对返回结果进行包装，重复劳动。



### 高级写法

借助SpringBoot提供的 `ResponseBodyAdvice` ，不要对每个接口都进行包装

> ResponseBodyAdvice的作用：拦截Controller方法的返回值，统一处理返回值/响应体，一般用来统一返回格式，加解密，签名等等。

编写一个实现类：

~~~java
@RestControllerAdvice
public class ResponseAdviceService implements ResponseBodyAdvice<Object> {

    @Resource
    private ObjectMapper objectMapper;

    /**
     * 是否支持advice功能
     * @param returnType
     * @param converterType
     * @return
     */
    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        return true;
    }

    /**
     * 处理返回结果
     * @param body
     * @param returnType
     * @param selectedContentType
     * @param selectedConverterType
     * @param request
     * @param response
     * @return
     */
    @SneakyThrows
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // 处理字符串类型数据
        if (body instanceof String) {
            return objectMapper.writeValueAsString(DataResult.success(body));
        }

        // 判断返回类型是否已经封装
        if (body instanceof DataResult) {
            return body;
        }
        return DataResult.success(body);
    }
}
~~~

String不能直接包装，必须额外处理



### 定义全局异常处理

~~~java
@RestControllerAdvice
@Slf4j
public class RestExceptionHandler {

    /**
     * 监控系统异常
     * @param e
     * @return
     */
    @ExceptionHandler(Exception.class)
    public DataResult handlerException(Exception e){
        log.error("handleException...{0}",e);
        return DataResult.getResult(BaseResponseCode.SYSTEM_ERROR);
    }

    /**
     * 监控自定义异常
     * @param e
     * @return
     */
    @ExceptionHandler(BusinessException.class)
    public DataResult handlerBusinessException(BusinessException e){
        log.error("BusinessException");
        return DataResult.getResult(e.getCode(),e.getMsg());
    }

    /**
     * 监控校验器异常
     * @param e
     * @return
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public DataResult handlerMethodArgumentNotValidException(MethodArgumentNotValidException e){
        List<ObjectError> allErrors = e.getBindingResult().getAllErrors();
        log.error("handlerMethodArgumentNotValidException AllErrors{} MethodArgumentNotValidException{}",e.getBindingResult().getAllErrors(),e);
        String msg=null;
        for (ObjectError error : allErrors) {
            msg = error.getDefaultMessage();
            break;
        }
        return DataResult.getResult(10001,msg);
    }

    /**
     * 监控无权限异常
     * @param e
     * @return
     */
    @ExceptionHandler(UnauthorizedException.class)
    public DataResult unauthorizedException(UnauthorizedException e){
        log.error("UnauthorizedException:{0}",e);
        return DataResult.getResult(BaseResponseCode.NO_AUTHORITY);
    }

}
~~~

> `RestControllerAdvice`，RestController的增强类，可用于实现全局异常处理器
>
> `@ExceptionHandler`,统一处理某一类异常，从而减少代码重复率和复杂度，比如要获取自定义异常可以`@ExceptionHandler(BusinessException.class)`
>
> `@ResponseStatus`指定客户端收到的http状态码



### 测试类

~~~java
@RestController
@RequestMapping("/test")
@Api(tags = "测试统一返回类")
public class TestController {

    @GetMapping("test1")
    private String test1() {
        return "hello world";
    }

    @GetMapping("test2")
    private Integer test2() {
        return 111;
    }

    @GetMapping("test3")
    private void test3() {
        throw new BusinessException(BaseResponseCode.NO_AUTHORITY);
    }

    @GetMapping("test4")
    private DomicileRespVo test4() {
        User user = new User();
        user.setName("木子李");
        user.setGender("男");
        user.setIdcard("123456");
        user.setAddress("安徽合肥");
        return respVo;
    }

}
~~~

