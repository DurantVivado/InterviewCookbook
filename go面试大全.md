# Go面试大全

推荐：官方FAQ

https://golang.org/doc/faq

### 什么是闭包？

闭包是引用了自由变量的函数。简单的说：

函数 + 引用环境 = 闭包

### new和make的区别？

new只用于分配内存，返回一个指向地址的指针。而make只可用于slice,map,channel的初始化,返回的是引用。

```
var a *int
a = new(int)
*a = 46
fmt.Println(*a)
```



### golang的内存管理的原理清楚吗？



1. Go什么时候发生阻塞？阻塞时，调度器会怎么做。

2. GMP中状态流转。

3. 如果有一个G一直占用资源怎么办。

4. 若干各线程一个线程发生OOM(Out of memory)会怎么办。

5. 怎么调试go？

6. 项目中错误处理怎么做？有统一的错误处理吗？

   

   

   

7. 如果若干个g，有一个panic会怎么做？

8. defer可以捕获g的子g吗？

9. 会自定义error吗？

10. gRPC是什么？

11. gRPC gateway怎么做？

12. proto文件是怎么管理？

13. 服务发现怎么做的？

14. ETCD

15. GIN错误处理用过吗？怎么做参数校验？

16. 中间件用过吗？

17. Go解析Tag是怎么实现的？

18. 反射用过吗？

19. Golang锁机制用过吗？锁有哪几种模式？

20. Channel用过吗？有什么坑？

不能

25. 数据库锁？MySQL锁。分布式锁？
26. Redis用的什么模式？主从模式和集群模式有什么区别？
27. golang的锁机制了解吗？Mutex锁有哪几种模式？Mutex锁底层了解过吗？
28. 持久化怎么做的？
29. MySQL用的ORM吗？

---

编程：

1. 实现使用字符串函数名，调用函数。

思路：采用反射实现。

```go
package main
import (
	"fmt"
    "reflect"
)

type Animal struct{
    
}

func (a *Animal) Eat(){
    fmt.Println("Eat")
}

func main(){
    a := Animal{}
    reflect.ValueOf(&a).MethodByName("Eat").Call([]reflect.Value{})
    
}
```



2. 负载均衡算法。

有一致性hash等





