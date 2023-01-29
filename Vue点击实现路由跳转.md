```vue
<div @click="gotoMenu">按钮</div>
```

 

**实现跳转**

```vue
methods: {
    gotoMenu(){
      //跳转到上一次浏览的页面  this.$router.go(-1)
      this.$router.go(-1)

      //指定跳转的地址 this.$router.replace('/menu')
      this.$router.replace('/menu')

      //指定跳转的路由的名字下 this.$router.replace({name:'menu'})
      this.$router.replace({name:'menu'})

      //通过push进行跳转
      this.$router.push('/menu')
    }
  },
}
```