---
title: Vue3调试环境准备
cover: "https://images.unsplash.com/photo-1573865526739-10659fec78a5?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxzZWFyY2h8NHx8YW5pbWFsJTIwbG92ZXxlbnwwfHwwfHw%3D&auto=format&fit=crop&w=500&q=60"
categories: 
     - Vue
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


{%  img
    "https://lh3.googleusercontent.com/awPtNabxVuQctH7Ryzi8EXO2guO8S2dv3l0alFURyzQlX3P5EmZEkWdou6iV9npg3sIIC2zSNs9OGqciikEg21WIE1YH5Pjyn8qD8lusU2ClDiR6-k2hIqAH1Ri4uvVL1XIF1BQh2vjwNIYw3qPai8whwkGM_2c1_IgVGPdvivWkE0U-yo7fTYOuqibBt2g9VNABeUr6ku8Y47TAXkGDCGGFnW79eGKucAno4ZSpI2RMZNtI-nqlIm-IB4THltcW6qW9xrsFnGqUQf7fC8J7cXPHfZDDpDRPwOpu_yRrBqxnSAZItV9iwIC4iE_hkYkbr-0Ds8djMJiruUOvJzy_058HURuyULrroL7x8hQ8HGcAVBAO1KVTt_Ay296iXq3yLbfAZ-wevthPn213DZf2Y4eow-0_p0PgAwPmviVmJevGx4zOvT5sCgxdEWMOddSgYbtZyaV1A4J86ckzNx7FodMIsQbTcH4a5CTY7_zBXXrbGQRsIx4Cl5P4KKSmIcyf34VvgelRnIVZvLp0oIWWYzZDrJGbur22Fb2SX0texLikWhqynO0kFLpAwj1VFs_in03NGsj4qEg4qhWnWsPuHDkWgWM02mVuNFqXdqrLOzb1KuLaJGnqHHtT6XCzwlSshUWRlI26pSZKE6gN18j_XGygEdVEVDJ3lcATw6G1LdFITP9JsJHIIqx6s-urofQ0n4_4wEQuauuaS-bMwgTO9mrJluA1uBYTJU2EGXqToAmf8pNQVwqCprQ-hA=w2420-h903-no?authuser=0"
    600
    600
%}

最后点击左侧的run，然后点击调试Vue调用即可调起调试的页面，也可以在源码中打断点进行调试

![Screenshot 2021-02-19 at 6.00.44 PM](./Screenshot 2021-02-19 at 6.00.44 PM.png)