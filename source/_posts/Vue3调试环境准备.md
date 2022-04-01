---
title: Vue3调试环境准备
cover: "https://images.unsplash.com/photo-1573865526739-10659fec78a5?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxzZWFyY2h8NHx8YW5pbWFsJTIwbG92ZXxlbnwwfHwwfHw%3D&auto=format&fit=crop&w=500&q=60"
---

# Vue3调试环境准备

## 1.克隆Vue3的代码

```javascript
git clone https://github.com/vuejs/vue-next.git
```

## 2.安装依赖和打包

```
yarn
yarn dev
```

yarn dev启动rollup将代码打包生成为vue.global.js，文件位置如图所示

![image (1)](./image (1).png)

### 3.新建demo页面，并在文件中引入vue.global.js文件

![Screenshot 2021-02-19 at 5.36.13 PM](./Screenshot 2021-02-19 at 5.36.13 PM.png)

此处贴上我的composition.html页面

```javascript
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <script src="../../packages/vue/dist/vue.global.js"></script>
</head>

<body>
    <div id="event-handling" class="demo">
        <p>{{ message }}</p>
        <button v-on:click="reverseMessage">Reverse Message</button>
    </div>
</body>
<script>
    const EventHandling = {
        data() {
            return {
                message: 'Hello Vue.js!'
            }
        },
        methods: {
            reverseMessage() {
                this.message = this.message
                    .split('')
                    .reverse()
                    .join('')
            }
        }
    }

    Vue.createApp(EventHandling).mount('#event-handling')
</script>

</html>
```

## 4.添加SourceMap文件

为了在浏览器能够查看源码和断点调试，可在rollup.config.js文件的createConfig函数中添加如下的命令，表示打包的时候会生成sourcemap文件用来记录函数的位置。

```
output.sourcemap = true
```

然后在在tsconfig.json中配置sourcemap输出

```
{
  "compilerOptions": {
    "baseUrl": ".",
    "outDir": "dist",
    // 修改sourcemap文件的配置
    "sourceMap": true,
    ...
}
```

完成以上操作即可实现在浏览器断点调试了，如果想在vscode进行断点调试可参考以下操作

## 5.修改.vscode的文件夹下面launch.json文件

添加如下配置，并修改file的路径为html demo文件的路径

![Screenshot 2021-02-19 at 5.56.53 PM](./Screenshot 2021-02-19 at 5.56.53 PM.png)

最后点击左侧的run，然后点击调试Vue调用即可调起调试的页面，也可以在源码中打断点进行调试

![Screenshot 2021-02-19 at 6.00.44 PM](./Screenshot 2021-02-19 at 6.00.44 PM.png)