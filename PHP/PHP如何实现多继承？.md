## PHP如何实现多继承？
### 1.trait是啥？
PHP可以通过trait实现多继承，trait看上去既像接口又像类，其实都不是，trait可以看做类的实现，可以混入一个或多个现有的类中，其作用有两个：1.表明类可以做什么；2.提供模块化实现。trait是一种代码复用技术，为PHP的单继承限制提供了一套灵活的代码复用机制；
### 2.PHP版本要求？
PHP5.4开始引入trait；
### 3.trait如何使用？
  1. 先声明一个trait；
  2. 在类中使用use将该trait引入；
### 4.trait的优先级？
自身的方法>trait的方法>继承的方法；
```php
<?php
trait HelloWorld {
    public function sayHello() {
        echo 'Hello World!';
    }
}
 
class TheWorldIsNotEnough {
    use HelloWorld;
    public function sayHello() {
        echo 'Hello Universe!';
    }
}
 
$o = new TheWorldIsNotEnough();
$o->sayHello();//输出是 Hello Universe!


//还有一点需要注意的是：多个trait的使用
trait Hello {
    public function sayHello() {
        echo 'Hello ';
    }
}
 
trait World {
    public function sayWorld() {
        echo 'World';
    }
}
 
class MyHelloWorld {
    use Hello, World;
    public function sayExclamationMark() {
        echo '!';
    }
}
 
$o = new MyHelloWorld();
$o->sayHello();
$o->sayWorld();
$o->sayExclamationMark();
?>
```
参考链接：https://blog.csdn.net/weixin_39445231/article/details/107271500
