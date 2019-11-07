
#### IOC VS 工厂模式
&emsp;&emsp;工厂模式和IOC作用一样，都是用来生产对象实例，但是工厂模式不方便之处在于场景扩展时需要首先修改模式内部代码，然后重新编译部署，比较麻烦，而Spring IOC是工厂模式和反射的结合，利用同一套创建实例的代码创建不同对象，动态生成对象，比如：

```java
// 工厂模式
interface Person {
    void say();
}
class Man implements Person {
    public void say() {
        System.out.println("I am man");
    }
}
class Woman implements Person {
    public void say() {
        System.out.println("I am woman");
    }
}
class Factory {
    public Person getPerson(String type) {
        if ("man".equals(type)) {
            return new Man();
        } else {
            return new Woman();
        }
    }
}
public class Test {
    public static void main(String[] args) {
         new Factory().getPerson("man").say();
         new Factory().getPerson("woman").say();
    }
}


// IOC方式
class Man {
    public void say() {
        System.out.println("I am man");
    }
}

class Woman {
    public void say() {
        System.out.println("I am woman");
    }
}

public class Test {
    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");
        Man man = (Man)ctx.getBean("man");
        Woman woman = (Woman)ctx.getBean("woman");
        man.say();
        woman.say();
    }

}

```
&emsp;&emsp;IOC很强大，低侵入性，但置步骤麻烦、生成对象的方式不直观、反射比正常生成对象在效率上慢一些。
