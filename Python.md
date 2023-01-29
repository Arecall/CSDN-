# Python学习

#### 输出类型

- 可以输出数字类

- 可以输出字符串

- 含有运算符的表达式

- 将数据输出到文件中

  - 指定的的盘符存在

  - 使用file=fq

    ```python
    fq=open('D:/text.txt','a+')#如果文件不存在就创建，存在就追加
    print('hello world',file=fq)
    fq.close()
    ```

#### 转义字符

- 转义字符串就是：反斜杠+要实现的转移字符首字母。
- 当字符串中包含反斜杠，单引号和双引号等有特殊用途时，必须使用反斜杠对这些字符串进行转义（转换一个含义）
  - 反斜杠：\\
  - 单引号：\'
  - 双引号：\"

- 当字符串包含换行，回车，水平制表或退格等无法直接表示的特殊字符时，也可以使用转义字符当字符串中包含换行，回车，水平制表或者退格等无法直接表示的特殊字符串时，也可以使用转义字符
  - 换行：\n
  - 回车：\r
  - 水平制表: \t
  - 退格： \b

```python
print('hello\nworld')
print('hello\tworld')
print('hello\rworld') #world将hello进行了覆盖
print('hello\bworld') #\b是退一格，将o删除

print('http:\\\\baidu.com')

print('Simon说：\'hello\'')

#原字符串，不希望字符串中的转义字符起作用就使用原字符串，就是在字符串之前加上r，或者R
print(r'hello\nworld') 
#最后一个字符串不能是反斜杠（\）
```

#### Python中的标识符和保留字

- 保留字

  - 有一些单词被赋予了特定的意义，这些单词在给你的任何对象起名字的时候都不能用

    ```python
    import keyword
    print(keyword.kwlist)
    ```

- 变量，函数，类，模块和其他对象的起的名称就是标识符

  规则：

  - 字母。数字，下划线。。。
  - 不能以数字开头
  - 不能是保留字
  - 严格区分大小写

#### 变量的定义和使用

```python
name='simon'
print(name)
print('标识',id(name))
print('类型',type(name))
print('值',name)
```

- 当多次赋值之后，变量名会指向新的空间

  ```python
  name = 'hello'
  name = 'simon'
  print(name)
  ```

#### 数据类型

- 常用的数据类型

  - 整数类型 -- int--98

    - 英文是integer，简写int，可以是正数也可以是复数和零

    - 整数的不同进制

      - 十进制  默认的进制（0~9）

        ```python
        print('十进制',118)
        ```

      - 二进制 以0b开头（0，1）

        ````python
        print('二进制',0b10101111)
        ````

      - 八进制  以0o开头（0，1，2，3，4，5，6，7）

        ```python
        print('八进制',0o176)
        ```

      - 十六进制  以0x开头（0~9，A~E）

        ```python
        print('十六进制',0x1EAF)
        ```

  - 浮点数类型  float ---3.14159

    - 由浮点数整数部分和小数部分组成

    - 浮点数存储不精准

      - 使用浮点数进行计算时，可能会出现小鼠位不确定的情况

        ```python
        n = 1.1
        s = 2.2
        print(s+n) #3.3000000000000003
        ```

      - 导入模块decimal

        ```python
        from decimal import Decimal
        print(Decimal('1.1')+Decimal('2.2'))
        ```

  - 布尔类型  bool --ture，flase

    - 用来表示真或者假的值

    - True是真，False是假

    - 布尔值可以转化为整数

      - True  ---  1

      - False   --- 0

        ```python
        f1 = True
        f2 = False
        print(f1,type(f1))
        print(f2,type(f2))
        
        print(f1+1)  #2 1+1的结果为2 True表示1
        print(f2+1)  #1 0+1的结果为1，False表示0
        ```

  - 字符串类型  str  ---  ’我在使用python

    - 字符串又被称为不可变的字符序列

    - 可以使用单引号，双引号，三引号(可以分多行显示)来定义 （''  ""  ''''''）

      ```python
      str1 = '人生苦短'
      str2 = "人生苦短"
      str3 = '''人生苦短，
      我用python'''
      str4 = """人生苦短，
      我用python"""
      
      print(str1,type(str1))
      print(str2,type(str2))
      print(str3,type(str3))
      print(str4,type(str4))
      ```

  - 数据类型转换

    - 将不同数据类型的数据拼接在一起

      ```python
      name = 'Simon'
      age =  22
      
      print(type(name),type(age)) #说明name和age数据类型不相同
      
      #将str类型与int类型进行连接，报错，解决方案：进行数据转换，将int类型转换成str类型
      #print('我叫'+name+ '今年'+age+'岁')
      print('我叫'+name+ '今年'+str(age)+'岁')
      
      print('------str()将其他的类型转成str类型------')
      a = 10
      b =198.8
      c = False
      print(type(a),type(b),type(c))
      print(str(a),str(b),str(c),type(str(a)),type(str(b)),type(str(c)))
      
      print('------将其他的类型转成int类型------')
      s1 = '128'
      s2 = 98.7
      s3 = '67.33'
      s4 = True
      s5 = 'hello'
      print(type(s1),type(s2),type(s3),type(s4),type(s5))
      
      print(int(s1),type(int(s1)))  #将str转成int类型，字符串为数字串
      print(int(s2),type(int(s2)))  #将float转成int类型，截取整数部分，舍掉小数部分
      # print(int(s3),type(int(s3))) 将str转成int类型，报错，因为字符串为小数串
      print(int(s4),type(int(s4)))
      #print(int(s5),type(int(s5)))  #将str转成int类型，字符串必须为数字串（整数），非数字串不允许转换
      
      print('------将其他的类型转成float类型------')
      s1 = '128.92'
      s2 = '76'
      s3 = True
      s4 = 'hello'
      s5 =98
      print(type(s1),type(s2),type(s3),type(s4),type(s5))
      print(float(s1),type(float(s1)))
      print(float(s2),type(float(s2)))
      print(float(s3),type(float(s3)))
      #print(float(s4),type(float(s4)))  字符串中的数据如果是非数字串，则不允许转换
      print(float(s5),type(float(s5)))
      ```

  - 注释

    ```python
    #输出 （单行）
    
    ''' 我是多行注释'''
    
    中文编码声明注释
    #coding:gbk 
    ```

#### Python的输入函数

###### 	input函数

- 作用：接收来自用户的输入

- 返回值类型：输入值的类型为str

- 值的存储：使用 = 对输入的值进行存储

  ```python
  a = input('输入第一个数值')
  a = int(a) # 将转换后的结果储存在a中
  b = input('第二个数值')
  b = int(b)
  print(type(a),type(b))
  print(a+b)
  
  #直接转化
  a = int(input('输入第一个数值'))
  b = int(input('第二个数值'))
  print(type(a),type(b))
  print(a+b)
  ```

  

#### Python的运算符

- 算数运算符

  - 标准算数运算符

  - 取余运算符

  - 幂运算符

    ```python
    print(1+1)
    print(1-1)
    print(2*4)
    print(1/2)
    print(11//2) # 整除运算
    print(11%2)  # 取余运算
    print(2**2)  # 幂运算
    ```

    

- 赋值运算符

  - 支持顺序： 右  👉 左

    ```python
    #从右到左
    a = 3+7
    print(a)
    ```

  - 支持链式赋值：a=b=c=20

    ```python
    #链路赋值
    b=c=d=20
    print(b,id(b))
    print(c,id(c))
    print(d,id(d))
    ```

  - 支持参数赋值： +=，-=，/=，//=，%=

    ```python
    a=20
    a+=20
    print(a,type(a))
    a-=10
    print(a)
    a*=2
    print(a,type(a))
    a/=3
    print(a,type(a))
    a//=3
    print(a,type(a))
    a%=3
    print(a)
    ```

  - 支持系类解包赋值：a,b,c=20，30，40

    ```python
    print('---------解包赋值----------')
    a,b,c=10,20,30
    print(a,b,c)
    #左右必须一一对应
    
    print('---------交换两个变量的值--------')
    a,b=10,20
    print('before',a,b)
    a,b=b,a
    print('after:',a,b)
    ```

- 比较运算符

  - ```
    >,<,>=,<=,!=
    ```

  - ```
    ==
    对象valuede 比较
    ```

  - ```python
    is,is not
    对象的id的比较
    
    a =10
    b =10
    print(a==b) #True  说明 a与b 的value相等
    print(a is b) #True 说明 a与b 的id标识 相等
    #value相同，id不同
    list1=[11,22,33]
    list2=[11,22,33]
    print(list1==list2)
    print(list1 is list2) #False
    print(id(list1))
    print(id(list2))
    #id是不相等的
    print(list1 is not list2)  #True
    ```

- 布尔运算符

  - and

    ```python
    #and有一个False的情况，则输出False
    a,b=1,2
    print(a==1 and b==2)
    print(a==1 and b<2)
    ```

  - or

    ```python
    print('------or------')
    #有一个True，则结果为True
    print(a==1 or b==2)
    print(a==1 or b!=2)
    print(a!=1 or b!=2)
    ```

  - not

    ```python
    print('-------not--------')
    #相反
    f =True
    f2 = False
    print(not f)
    print(not f2)
    ```

  - in

  - not in

    ```python
    print('-------in not in---------')
    s = 'hello Simon'
    print('s' in s)
    print('s' not in s)
    ```

- 位运算符

#### 运算的优先级

###### 	将数据转成二进制进行计算

- 位与& ：对应数都是1，结果数位才是1，否则为0

- 位或 | ： 对应数位都是0，结果数位才是0，否则为1

- 左移位运算符<< : 高位溢出舍弃，低位补0

- 又移位运算符>> ：低位溢出舍弃，高位补0

  ```python
  print(4&8)  # 按位与& ，同为1时结果为1
  
  print(4|8) #按位与 | ，同为0时结果为0
  
  print(4<<1)  #按位左移运算符，低位补0
  print(4<<2)
  
  print(4>>1)  #右移位，高位补0
  ```

  ![image-20200922111841531](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200922111841531.png)

  先算 算数运算，再算位运算，然后算比较运算（True / False），之后算布尔运算（and ，or）,最后是赋值运算

  `（）--算数运算--位运算--比较运算--布尔运算--赋值运算`

#### 程序的组织结构_顺序结构

1. 程序的组织机构

   1. 计算机的流程控制
      - 顺序结构
      - 选择结构  if 语句
      - 循环结构  while语句   for-in语句

2. 顺序结构

3. 对象的布尔值

   - python一切皆对象，所有对象都有一个布尔值

   - 获取对象的布尔值（使用内置函数bool）

   - 以下的对象都是布尔值False

     - False

       `print(bool(False))`

     - 数值（）

       ```python
       print(bool(0))
       print(bool(0.0))
       print(bool(None))
       ```

     - 空字符串

       ```python
       print(bool(''))
       print(bool(""))
       ```

     - 空列表

       ```python
       print(bool([]))  #空列表
       print(bool(list())) #空列表
       ```

     - 空元组

       ```python
       print(bool(()))  #空元组
       print(bool(tuple())) #空元组
       ```

     - 空字典

       ```python
       print(bool({}))
       print(bool(dict()))
       ```

     - 空集合

       ```python
       print(bool(set()))  #空集合
       ```

       

4. 分支结构

   - 单分支if结构

     ```python
     money =1000
     s = int(input('请输入取款金额'))
     #判断余额是否充足
     if money >= s:
         money=money-s
         print('取款成功，余额为：',money)
     ```

   - 双分支if...else结构

     ```python
     num = int(input('请输入一个整数'))
     #条件判断
     if num%2 ==0:
         print(num,'是偶数')
     else:
         print(num,'是奇数')
     ```

   - 多分支if...elif...else结构

     ```python
     score=int(input('请输入你的成绩'))
     if score >=90 and score<=100:
         print('您的成绩优秀哦')
     elif score >=80 and score<90:
         print('您的成绩良好')
     elif score >=70 and score<80:
         print('您的成绩一般')
     elif score >=60 and score<70:
         print('您的成绩已及格')
     #python 可以这样写大小比较
     elif 0<=score<60:
         print('您的成绩不合格，还需要加油哦！')
     else:
         print('您的输入有误，请重新输入！！')
     ```

   - if语句的嵌套

     ```python
     answer = input('是会员吗？(y/n)')
     moenry = int(input('输入金额：'))
     if answer=='y':
         if moenry>=200:
             print('此次消费打八折，付款金额为：',moenry*0.8)
         elif moenry>=100:
             print('此次消费打九折，付款金额为：',moenry*0.9)
         else:
             print('不打折，付款金额为',moenry)
         print('会员消费')
     else:
         if moenry>=200:
             print('此次消费打九五折，应付款：',moenry*0.95)
         else:
             print('不打折，应付款：',moenry)
         print('非会员消费')
     ```

   - 条件表达式

     ```python
     num_a = int(input('请输入第一个整数：'))
     num_b = int(input('请输入第二个整数：'))
     print(str(num_a)+'大于等于'+str(num_b) if num_a>=num_b else str(num_a)+'小于'+str(num_b))
     ```

5. pass空语句

   - 语句什么都不做，只是一个占位符，用在语法上需要语句的地方。

   - 什么时候使用：先搭建语法结构，还没想好代码怎么写的时候

   - 哪些语句一起使用

     - if语句的条件执行体
     - for-in语句的循环题
     - 定义函数的函数体

     ```python
     answer = input('您是会员吗？')
     if answer == 'y':
         pass
     else:
         pass
     ```

#### 内置函数rangge（）

1. range（）函数

   - 用于生成一个整数序列

   - 创建range对象的三种方式

     - range（stop）--->  创建一个（0，stop）之间的整数序列，步长为1

     - range（start，stop）---> 创建一个（start，stop）之间的整数序列，步长为1

     - range（start，stop，step）---> 创建一个（start，stop）之间的整数序列，步长为step

       ```python
       '''第一种创建方式，只是一个参数（小括号中只给一个数）'''
       r = range(10) #[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]，默认从0开始，默认相差1称为步长
       print(r) #range(0,10)
       print(list(r)) #用户查看range对象中的整数序列     --->list是列表的意思
       
       '''第二种创建方式，给了两个 参数（小括号给两个数）'''
       r = range(1,10) #指定了起始值，从1开始，到10结束（不包含10），默认步长为1
       print(list(r)) # [1, 2, 3, 4, 5, 6, 7, 8, 9]
       
       '''第三种创建方式，给三个参数（小括号给三个数）'''
       r = range(1,10,2)
       print(list(r)) #[1, 3, 5, 7, 9]
       ```

     - 返回值是一个迭代器对象

     - range类行的优点：不管range对象表示的整数序列有多长，所有的range对象占用的内存空间是相同的，因为仅仅需要存储在start，stop和step，只有当用到range对象时，才会去计算序列相关元素

     - in 与not in判断整数序列中是否存在（不存在）指定的整数

       ```python
       r = range(1,10,2)
       print(10 in r) #False
       print(9 in r) #True
       print(9 not in r) #False
       print(10 not  in r) #True
       ```

#### 循环结构

1. 循环分类

   - while：while是判断N+1次，条件为True执行N次

     ```python
     a = 1
     while a<10:
        # a+=1
         print(a)
         a+=1
     ```

     - 四部循环法：
       1. 初始化变量
       2. 条件判断
       3. 条件执行体（循环体）
       4. 改变变量

   - for-in

     - for in循环

       - in表达从（字符串，序列等）中依次取值，又称为遍历
       - for-in遍历的对象必须是可迭代对象

     - for-in的语法结构

       - for 自定义的变量 in 可迭代对象：

         ​	循环体

       - 循环体内不需要访问自定义变量，可以将自定义变量替代为下划线

   - break

     ```python
     print('---------for--------')
     for item in range(3):
         pwd =input('请输入密码：')
         if pwd == '8888':
             print('密码正确')
             break
         else:
             print('密码错误')
     
     print('----------while------------')
     a =0
     while a<3:
         pwd = input('请输入密码：')
         if pwd =='888':
             print('密码正确')
             break
         else:
             print('ooo')
         a+=1
     ```

   - continue：结束本次循环，执行下一次的循环

     ```python
     for item in range(1,51):
         if item%5 !=0:
             continue
     
         print(item)
     ```

   - 九九乘法表

     ```python
     print('------------')
     for i in range(1,10):
         for j in range(1,i+1):
             print('*',end='\t')
         print()
     
     
     print('-'*20+'9*9'+'-'*20)
     for i in range(1,10):
         for j in range(1,i+1):
             print(i,'*',j,'=',i*j,end='\t')
         print()
     
     ```

#### 列表

- 列表的特点

  - 列表元素按顺序有序排列

    ```python
    lst = ['hi','Simon',000]
    print(lst[0],lst[1])
    ```

  - 索引映射唯一数据

  - 列表可以储存重复的数据

  - 任意数据类型混存

  - 根据需要动态分配和回收内存

- 列表的查询

  - 获取列表中指定元素的索引

    ```python
    lst = ['hello','Simon',99,'hello']
    print(lst.index('hello')) #如果列表中有相同的元素只返回列表中相同元素的第一个元素索引
    
    print(lst.index('hello',1,4))
    ```

  - 获取列表中的单个元素：

    ```python
    print(lst[1])
    ```

    

- 获取列表的多个元素

  - 语法格式

    列表名[start ： stop： step] 左闭右开

    ```python
    lst = [10,20,30,40,50,60,70,80]
    
    print(lst[1:6:1]) #[20, 30, 40, 50, 60]
    #步长不写，默认是1
    print(lst[1:6]) 
    print('beforce:',id(lst))
    print('after:',id(lst))
    
    '''start:1 stop:6  step:2'''
    print(lst[1:6:2])
    
    print('----------step负数------------')
    print(lst[7:0:-1]) #[80, 70, 60, 50, 40, 30, 20]
    ```

- 列表的增删查改

  列表的增加

  ```python
  #向列表的末尾增加一个元素
  lst = [10,20,30]
  print(lst,id(lst))  #[10,20,30]
  lst.append(100)
  print(lst,id(lst)) #[10, 20, 30, 100]
  #向列表末尾增加多个元素
  lst2 = ['hello','Simon']
  lst.extend(lst2)
  print(lst) #[10, 20, 30, 100, 'hello', 'Simon']
  #在任意位置增加一个元素
  lst.insert(1,15)
  print(lst)
  #在任意位置上添加多个元素
  lst3 = ['moring','jd',100000]
  lst[2:] = lst3
  print(lst) #[10, 15, 'moring', 'jd', 100000]
  ```

  列表的删除

  ```python
  lst = [10,20,30,40,50,60,30]
  lst.remove(30) #删除重复元素的第一个元素
  print(lst)
  
  #pop()根据索引移除元素
  lst.pop(1)
  print(lst)
  lst.pop()
  print(lst) #如果为空，则删除最后一个
  
  print('-----------切片操作-删除至少一个元素，将产生新的列表对象----------------')
  new_lst = lst[1:3]
  print('bofore:',lst)
  print('after:',new_lst)
  
  '''不产生新的列表对象，而是删除原列表的内容'''
  lst[1:3]= []
  print(lst)
  
  '''清除列表所有的元素'''
  lst.clear()
  print(lst)
  
  '''del 语句将列表删除'''
  del lst
  ```

  列表元素的修改

- 为指定索引的元素赋予一个新值

- 为指定的切片赋予一个新值

  ```python
  lst = [10,20,30,40]
  #一次修改一个值
  lst[2] = 100
  print(lst)
  
  #一次修改多个
  lst[1:3] = [200,300,400]
  print(lst)
  ```

  列表元素的排序操作

- 常见的两种方式

  - 调用sort（）方法，对列表中所有的元素默认按照从小到大的顺序排序，可以 指定reverse=True，进行降序排序(原列表上操作)

    ```python
    lst = [30,50,10,99,58]
    print(lst,id(lst))
    #默认升序
    lst.sort()
    print(lst,id(lst))
    print('------降序------')
    lst.sort(reverse=True)
    print(lst)
    ```

  - 调用内置函数sorted（），可以指定reverse=True，进行降序排序（产生新的列表），原列表不发生改变。

    ```python
    print('-------内置函数排序-------')
    lst = [30,50,10,99,58]
    new_lst=sorted(lst) #默认是升序
    print(lst)
    print('------指定关键词，进行降序------')
    dec_lst = sorted(lst,reverse=True)
    print(dec_lst)
    ```

  列表生成式

- 简称“生成列表的公式”

  - 语法格式：[i*i for i in range(1,10)]

    ```python
    lst = [i for i in range(1,10)]
    print(lst) #[1, 2, 3, 4, 5, 6, 7, 8, 9]
    print('-----------')
    lst = [i*i for i in range(1,10)]
    print(lst) #[1, 4, 9, 16, 25, 36, 49, 64, 81]
    
    print('----------列表2，4，6，8，10-----------------')
    lst1 = [i*2 for i in range(1,6)]
    print(lst1)
    ```

#### 字典

- 什么是字典

  - Python内置的数据结构之一，与列表一样是一个可变序列

  - 以键值对的方式存储数据，字典是一个无序的序列

    ```python
    '''字典的创建'''
    # 使用 {} 创建字典
    scores = {'张三':100,'李四':90,'王宇':101}
    print(scores,type(scores))
    
    '''内置函数dict（）'''
    stu = dict(name ='jack',age = 30)
    print(stu)
    
    '''空字典'''
    d = { }
    print(d)
    ```

- 字典的原理

- 字典的创建与删除

- 字典的查询操作

  ###### 字典中元素的获取

  ```python
  scores = {'张三':100,'李四':90,'王宇':101}
  
  '''获取字典元素'''
  #第一种方式
  print(scores['张三'])
  #print(scores['sixhi']) KeyError
  
  #第二中方式，使用get（）方法
  print(scores.get('张三'))
  print(scores.get('xishi')) #None
  print(scores.get('xishi','不存在')) #当查找不存在就输出默认值
  ```

  

- 字典元素的增，删，该操作

  key的判断

  ```python
  scores = {'张三':100,'李四':90,'王宇':101}
  
  print('张三' in scores)
  print('张三' not in scores)
  
  #delete
  del scores['张三'] #删除指定的键值对
  print(scores)
  
  #scores.clear() #清空字典元素
  
  scores['柽柳']=110 #新增元素
  print(scores)
  #修改
  scores['柽柳']=190
  print(scores)
  ```

  

- 字典推导式

