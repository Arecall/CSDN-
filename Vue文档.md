### Vue文档

##### 定义数据对象

1. data：用于定义属性
2. methods：用于定义函数，通过return来返回函数。
3. {{ }}用于输出对象属性和返回值

##### Vue是一个渐进式的框架

- vue作为应用中的一部分嵌入其中，体验更加丰富。
- 实现更多的业务逻辑，vue有丰富的生态及核心库。

##### vue有很多的特点和web开发常见的高级功能

- 解耦试图和数据
- 可复用的组件
- 前端路由技术
- 状态管理
- 虚拟DOM

##### NPM安装

```
npm install vue
```

##### let（变量） const（常量）

![image-20200904175144768](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200904175144768.png)

###### vue列表展示

- ​	v-for进行遍历数组

  ![image-20200904181326891](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200904181326891.png)

###### 计数器

- ​	v-on：监听
- ​	语法糖：简写
- ​	methods，用于定义语法。

![image-20200904182823469](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200904182823469.png)

##### MVVM（维基百科地址：https://zh.wikipedia.org/wiki/MVVM）

![image-20200904183754460](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200904183754460.png)

**MVVM**（**Model–view–viewmodel**）是一种软件[架构模式](https://zh.wikipedia.org/wiki/架构模式)。

- view：
  - 视图层
  - 在前端开发中，通常就是DOM层
  - 主要的作用是给用户展示各种信息
- Model：
  - 数据层
  - 数据可能是开发固定写死的，更多的是来自服务器请求，从网络下载的数据
  - 在计数器实例中，就是从后面抽取的object，当然数据也有一定的复杂性。
- VueModel：
  - 视图层
  - 视图模型层是View和Model沟通的桥梁
  - 一方面它实现了databinding，也就是数据绑定，将Model的改变为实时反馈到View中
  - 另一方面它实现了DOM Listener，也就是DOM监听，当DOM发生一些事件（点击，滚动，touch）时，可以监听到，并在需要的情况下改变对应的Data。

### 创建Vue实例传入options

- el
  - 类型：string| HTMLElement
  - 作用：决定之后Vue实例会管理哪一个DOM
- data
  - 类型：Object| Function
  - 作用：Vue实例对应得数据类型
- methods：
  - 类型：{[key:string]: Function}
  - 作用：定义属于Vue的一些方法，可以在其他地方调用，也可以在指令中使用。

### vue的生命周期

- 

### 模板语法

- #### 插值操作

  - Mustache语法

    {{  }}

    ![image-20200905162205847](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200905162205847.png)

  - v-once：只会加载第一次数据，不会跟随数据进行改变。

    ![image-20200905162657485](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200905162657485.png)

  - v-html：解析url

    ![image-20200905163503326](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200905163503326.png)

  - v-text：会覆盖标签里面其他文字

    ![image-20200905164118541](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200905164118541.png)

  - v-pre：会原封不动的展示里面的内容

    ![image-20200905164512216](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200905164512216.png)

  - v-cloak： cloak（斗篷）

    数据未渲染之前，进行隐藏操作。

    ![image-20200905170129254](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200905170129254.png)

- #### 动态绑定

  - v-bind:  

    ![image-20200906093413641](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200906093413641.png)

    - v-bind-class  对象语法

      v-bind:class="{ }" //{}里表示是对象，{ key：value}键值对

      ![image-20200906101529113](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200906101529113.png)
      
    - const使用
    
      在es6开发中，优先使用const，只有需要改变某个标识符时才使用let
  
- #### 时间监听

  - v-on介绍

    - 作用：绑定事件监听器
    - 缩写：@
    - 预期：Function | Inline Statement | Object
    - 参数：event

  - 进行事件监听的时候，调用的方法不传递参数，方法后面的（）可以省略不写。

    ```javascript
    <div id="app">
      <h2>{{counter}}</h2>
    <!--  语法糖-->
      <button @click="increment">+</button>
    
      <button v-on:click="decrement">-</button>
    </div>
    <script src="../js/vue.js"></script>
    <script>
      const app = new Vue({
        el: '#app',
        data: {
          counter: 0
        },
        methods: {
          increment(){
            this.counter++;
            alert('+1')
          },
          decrement(){
            this.counter--
            alert('-1')
          }
        }
        })
    </script>
    ```

    
