# MySQL语法：

1. ###### Windows本地启动

   ```
   mysqld --console启动
   
   密码
   mysql -uroot -p
   
   关闭
   mysqladmin -uroot shudown　　
   或　　net stop mysql　　
   ```

2. ###### 数据库查看

   ```
   show databases; #查看数据库
   use 数据库名
   show tables;  进入表格
   ```

3. ###### 数据类型

   ![image-20200713112318316](C:\Users\11563\AppData\Roaming\Typora\typora-user-images\image-20200713112318316.png)

4. ###### 外键语法

   其中，外键约束的名称fk_class_id可以任意，FOREIGN KEY (class_id)指定了class_id作为外键，REFERENCES classes (id)指定了这个外键将关联到classes表的id列（即classes表的主键）。

   ```mysql
   ALTER TABLE students
   ADD CONSTRAINT fk_class_id
   FOREIGN KEY (class_id)
   REFERENCES classes (id);
   ```

   要删除一个外键约束，也是通过`ALTER TABLE`实现的：

   ```mysql
   ALTER TABLE students
   DROP FOREIGN KEY fk_class_id;
   ```

   