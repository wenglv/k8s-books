##### Library代码结构介绍

共享库的目录结构如下:

```bash
(root)
+- src                     # Groovy source files
|   +- org
|       +- foo
|           +- Bar.groovy  # for org.foo.Bar class
+- vars
|   +- devops.groovy          # for global 'foo' variable
|   +- foo.txt             # help for 'foo' variable
```

`src` 目录应该看起来像标准的 Java 源目录结构。当执行流水线时，该目录被添加到类路径下。

`vars` 目录定义可从流水线访问的全局变量的脚本。 每个 `*.groovy` 文件的基名应该是一个 Groovy (~ Java) 标识符, 通常是 `camelCased`。 

##### Groovy基本语法介绍

新建Groovy项目

- 变量

  使用数据类型的本地语法，或者使用def关键字 

  ```groovy
  // Defining a variable in lowercase  
  int x = 5;
  
  // Defining a variable in uppercase  
  int X = 6; 
  
  // Defining a variable with the underscore in it's name 
  def _Name = "Joe"; 
  
  println(x); 
  println(X); 
  println(_Name); 
  
  ```

- 方法

  - 调用本地方法

    ```bash
    def sum(int a, int b){
        return a + b
    }
    
    println(sum(1,2))
    ```

  - 调用类中的方法

    ```bash
    # Hello.groovy
    package demo
    
    def sayHi(String content) {
        return ("hi, " + content)
    }
    
    
    
    # Demo.groovy
    import demo.Hello
    
    def demo() {
        return new Hello().sayHi("devops")
    }
    println(demo())
    
    
    
    # 级联调用
    # Hello.groovy
    package demo
    
    def init(String content) {
        this.content = content
        return this
    }
    
    def sayHi() {
        println("hi, " + this.content)
        return this
    }
    
    def sayBye() {
        println("bye " + this.content)
    }
    
    
    # Demo.groovy
    import demo.Hello
    
    def demo() {
        new Hello().init("devops").sayHi().sayBye()
    }
    
    demo()
    
    ```

- 异常捕获

  ```groovy
  def exceptionDemo(){
      try {
          def val = 10 / 0
          println(val)
      }catch(Exception e) {
          println(e.toString())
          throw e
      }
  }
  exceptionDemo()
  ```

- 计时器与循环

  ```groovy
  import groovy.time.TimeCategory
  
  
  use( TimeCategory ) {
      def endTime = TimeCategory.plus(new Date(), TimeCategory.getSeconds(15))
      def counter = 0
      while(true) {
          println(counter++)
          sleep(1000)
          if (new Date() >= endTime) {
              println("done")
              break
          }
      }
  }
  ```

- 解析yaml文件

  ```bash
  import org.yaml.snakeyaml.Yaml
  
  def readYaml(){
      def content = new File('myblog.yaml').text
      Yaml parser = new Yaml()
      def data = parser.load(content)
      def kind = data["kind"]
      def name = data["metadata"]["name"]
      println(kind)
      println(name)
  }
  readYaml()
  ```

  