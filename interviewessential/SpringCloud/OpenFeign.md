# OpenFeign

![1759916355664-3f6e5422-7fde-4fa7-8224-9e5e29770519.png](./img/890XFVGE38RORhCF/1759916355664-3f6e5422-7fde-4fa7-8224-9e5e29770519-684167.png)



拦截器是 请求拦截器和 相应拦截器



```java
@Component
public class TokenInceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate requestTemplate) {
        requestTemplate.header("token", UUID.randomUUID().toString());
    }
}
```



> 更新: 2025-10-13 08:56:17  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/tnxcaotb5i60uh6h>