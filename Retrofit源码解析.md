```java
public <T> T create(final Class<T> service) {
Utils.validateServiceInterface(service);
if (validateEagerly) {
  eagerlyValidateMethods(service);
}
//返回一个动态代理
return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
    new InvocationHandler() {
      private final Platform platform = Platform.get();

      @Override public Object invoke(Object proxy, Method method, Object... args)
          throws Throwable {
        // If the method is a method from Object then defer to normal invocation.
        if (method.getDeclaringClass() == Object.class) {
          return method.invoke(this, args);
        }
	//没什么卵用的代码
        if (platform.isDefaultMethod(method)) {
          return platform.invokeDefaultMethod(method, service, proxy, args);
        }
        ServiceMethod serviceMethod = loadServiceMethod(method);
        OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
        return serviceMethod.callAdapter.adapt(okHttpCall);
      }
    });
}
	此时，invoke的返回必然是一个Call，Call是Retrofit中对一个Request的抽象,由此，大家应该不难想象到loadMethodHandler(method).invoke(args); 这句代码应该就是去解析接口中传进来的注解，并生成一个OkHttpClient中对应的请求，这样我们调用searchResultsCall时，调用OkHttpClient走网络即可。确实，Retrofit的主旋律的确就是这样滴。
      //proxy对象就是你在外面调用方法的resetApi对象
      //method是RestApi中的函数定义，
      //据此，我们可以获取定义在函数和参数上的注解，比如@GET和注解中的参数
      //args,实际参数，这里传送的就是字符串"retrofit"

```