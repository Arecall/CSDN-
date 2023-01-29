1. 首先到node.js官网下载：https://nodejs.org/en/download/

   然后在cmd命令下执行 **npm install webpack -g** 安装webpack

   **webpack -v** 查看版本

2. 安装Vue cli

   ```
   关于旧版本
   ```

   ```
   Vue CLI 的包名称由 `vue-cli` 改成了 `@vue/cli`。 如果你已经全局安装了旧版本的 `vue-cli` (1.x 或 2.x)，你需要先通过 `npm uninstall vue-cli -g` 或 `yarn global remove vue-cli` 卸载它。
   ```

   可以使用下列任一命令安装这个新的包：

   ```bash
   npm install -g @vue/cli
   # OR
   yarn global add @vue/cli
   ```

   你还可以用这个命令来检查其版本是否正确：

   ```bash
   vue --version
   ```

  如果不能运行 npm run dev 则安装yarn