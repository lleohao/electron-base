>原文发表于 `https://lleohao.github.io/2017/09/02/如何搭建Electron开发环境/`
>
>这个项目结构是我在编写  [基于Electron 和 Angular 的七牛文件上传App](https://github.com/lleohao/qiniu-upload)  总结出来的
>
>本文主要介绍如何从零开始搭建高效的Electron开发环境, 主要内容如下：
>
>1. 通过合理的目录划分来组织代码
>2. 使用`npm script`简化开发
>3. 如何在渲染进程开发时使用热更新
>4. 如何在主进程开发时使用自动重启
>5. 如何在主进程开发时使用`Typescript`
>6. 如何打包和发布软件
>
>示例项目地址 https://github.com/lleohao/electron-base

> 发现 http://hao.jser.com/ 这个网站臭不要脸的转载文章



## 目录结构划分

### 初始化目录

首先按照常规的方法新建一个项目文件夹(这里我的示例文件夹叫做`electron-base`， 然后使用`npm init`初始化目录。

目前我们的开发目录如下：

```
electorn-base
├── .gitignore - git忽略文件
├── LICENSE - 开源协议
├── README.md - 文档
└── package.json - npm package
```



### 目录划分

Electron 的开发主要分为两个部分,  其中主进程(Main Process)主要负责打开页面和调用系统底层的资源等, 渲染进程(Renderer Process)则是一个普通的网页窗口.

两个进程的开发有着不同的开发方式, 主进程更像是传统`Node`的开发, 而渲染进程则是普通的前端开发. 同时它们之间又有着可以共用的部分(如辅助函数、数据模型等), 因此可以按照下面的方式划分

```
electorn-base
├── ... - 省略
└── src - 代码源文件
    ├── main - 主线程代码
    ├── renderer - 渲染线程
    └── shared - 公用代码
```



### Electron quick start

接下来运行`npm install electron -D`安装Electron，同时在`package.json`添加`main`字段， 这代表整个项目的入口文件，这里我们先设置为`src/main/main.js`.

顺便添加上两个文件

```javascript
# src/main.js
const { app, BrowserWindow } = require('electron')
const path = require('path')
const url = require('url')

let win

function createWindow() {
    win = new BrowserWindow({ width: 800, height: 600 })

    win.loadURL(url.format({
        pathname: path.join(__dirname, '../renderer/index.html'),
        protocol: 'file:',
        slashes: true
    }))

    // Open the DevTools.
    win.webContents.openDevTools()

    // Emitted when the window is closed.
    win.on('closed', () => {
        // Dereference the window object, usually you would store windows
        // in an array if your app supports multi windows, this is the time
        // when you should delete the corresponding element.
        win = null
    })
}

// This method will be called when Electron has finished
// initialization and is ready to create browser windows.
// Some APIs can only be used after this event occurs.
app.on('ready', createWindow)

// Quit when all windows are closed.
app.on('window-all-closed', () => {
    // On macOS it is common for applications and their menu bar
    // to stay active until the user quits explicitly with Cmd + Q
    if (process.platform !== 'darwin') {
        app.quit()
    }
})

app.on('activate', () => {
    // On macOS it's common to re-create a window in the app when the
    // dock icon is clicked and there are no other windows open.
    if (win === null) {
        createWindow()
    }
})
```

```html
<!-- src/renderer/index.html -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Hello World!</title>
  </head>
  <body>
    <h1>Hello World!</h1>
    We are using node <script>document.write(process.versions.node)</script>,
    Chrome <script>document.write(process.versions.chrome)</script>,
    and Electron <script>document.write(process.versions.electron)</script>.
  </body>
</html>
```

在根目录运行`electron .`(或者是`./node_modules/.bin/electron .`)启动程序

为了以后方便启动程序, 将这段命令添加到`package.json`中

```json
// package.json 部分内容
"main": "src/main/main.js",
"scripts": {
	"start": "./node_modules/.bin/electron ."
},
"devDependencies": {
	"electron": "^1.7.5"
}
```



## 开发渲染线程

渲染线程的开发跟普通的前端开发没有多大的区别, 为了开发的效率, 我们通常会选择一款前端开发框架, 这里我选择的是`Angular`, 当然也可以选择其他的框架, 只需要按照下文中的要求修改打包路径.



### 导入Angular(可选, 使用其他框架可以跳过)

这里我使用`Angular-cli`工具来初始化项目

1. 安装cli工具

    `npm install -g @angular/cli`

2. 初始化目录

    ` ng new electron-base -sd src/renderer -si -sg -st --routing true --styles scss ` 

3. 修改`.angular-cli.json`

    ```Json
    "apps": [{
      "root": "src/renderer",	// 源文件目录
      "outDir": "out/renderer", // 输出目录
      "baseHref": "./", // 解决打包后无法加载文件
      ...
    }]
    ```




### 如何在开发过程中进行代码热更新

前端开发中, 我们可以使用`webpack`享受到自动刷新、热更新等方便的功能， 那么在Electron的开发过程我们如何享受的到这些功能了？这里我们只需要简单的修改下`main.js`文件即可

```javascript
function isDev() {
    return process.env['NODE_ENV'] === 'development'
}

function createWindow() {
    win = new BrowserWindow({ width: 800, height: 600 })

    if (isDev()) {
        // 这里的url换成你所使用框架开发时的url
        win.loadURL('http://127.0.0.1:4200');
    } else {
        win.loadURL(url.format({
            pathname: path.join(__dirname, '../renderer/index.html'),
            protocol: 'file:',
            slashes: true
        }))
    }

    // Open the DevTools.
    win.webContents.openDevTools()

    // Emitted when the window is closed.
    win.on('closed', () => {
        // Dereference the window object, usually you would store windows
        // in an array if your app supports multi windows, this is the time
        // when you should delete the corresponding element.
        win = null
    })
}
```

开发时我们还是按照以前的方式启动一个`webpcak`服务器进行开发, Electron通过`HTTP`协议打开页面, 这样我们依旧可以享受到代码热更新等功能.

通过设置环境变量`NODE_ENV`来区分开发和生成环境, 在`package.json`中添加两个命令来方便开发

```json
"scripts": {
  "ng": "ng", // angular alias
  "start": "NODE_EBV=production ./node_modules/.bin/electron .", // 添加环境变量
  "dev:renderer": "ng serve" // 启动渲染线程开发服务器
},
```



### 打包渲染线程代码

开发完成后我们需要将前端的代码进行代码打包, 一个好的习惯是将代码的打包目录放置在项目的根目录中, 这里我将前端的打包目录设置在`out/renderer`中

Angular项目只需要修改`.angular-cli.json`中的`outDir`字段, 其他的框架可以自行修改. 

在`package.json`中添加打包命令

```Json
"scripts": {
  ....
  "build:renderer": "ng buidl --prod" // 打包渲染线程代码
},
```



## 开发主线程

主线程的开发如`Node`程序的开发没有多大的区别, 这里就不多赘述.

虽然`Node`对`ES6`的支持已经很完善了, 但更新的标准的支持就不怎么好, 这里我们可以使用`Babel`之类的工具进行来使用最新的语法.

这里我推荐使用`Typescript`, 优点主要有三个:

1. 静态检查, 毕竟是主线程的代码, 有点错误可就是程序直接崩溃的节奏
2. 自动提示, 这个不解释
3. 编译方便, 比起`Babel`的配置文件, `Typescript`的配置要简单的多





### 使用Typescript (不使用的可以跳过)

1. 安装`Typescript`

   运行`npm install typescript -D`

2. 添加配置文件, 在`src`目录下添加`tsconfig.main.json`文件

   ```json
   {
       "compilerOptions": {
           "outDir": "../out",  	// 输出目录, 同渲染线程放在一起
           "sourceMap": true,	 	// 调试时需要
           "moduleResolution": "node",
           "emitDecoratorMetadata": true,
           "experimentalDecorators": true,
           "target": "es6",     	// 输出代码版本, 由于在Node中运行, es6没问题
           "module": "commonjs",	// module 处理方式
           "typeRoots": [			// .d.ts 目录
               "../node_modules/@types"
           ],
           "lib": [				// 可选, 添加新的语法支持
               "es2017"
           ]
       },
      "exclude": [					// 排除渲染线程目录
           "renderer"
      ]
   }
   ```

3. 在`package.json`中添加开发和打包命令

   ```json
   "scripts": {
   ...
     "dev:main": "tsc -p ./src/tsconfig.main.json -w", // 开发
     "build:main": "tsc -p ./src/tsconfig.main.json"   // 打包
   }
   ```




### 主线程调试 (使用的编辑器是vscode)

1. 添加启动配置文件, 项目根目录新建`.vscode`文件夹,在其中新建`launch.json`

   ```json
   {
       // Use IntelliSense to learn about possible Node.js debug attributes.
       // Hover to view descriptions of existing attributes.
       // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
       "version": "0.2.0",
       "configurations": [{
           "type": "node",
           "request": "launch",
           "name": "Launch Program",
           "cwd": "${workspaceRoot}",
           "runtimeExecutable": "${workspaceRoot}/node_modules/.bin/electron",
           "program": "${workspaceRoot}/src/main/main.ts", // 你的主文件
           "sourceMaps": true,							
           "outFiles": [
               "${workspaceRoot}/out/**/*.js"	// 你的输出文件目录
           ],
           "env": {
               "NODE_ENV": "development"
           }
       }]
   }
   ```

2. 使用组合键`ctrl + f5`启动程序

3. 在文件中添加断点进行调试



### 主线程开启自动刷新 (可选)

我们的渲染线程可以做到代码变更后自动刷新页面, 在主线程的开发中我们可以使用 [nodemon](https://github.com/remy/nodemon) 来实现同样的功能

1. 安装`nodemon`

   `npm install nodemon -D`

2. 修改启动命令

   ```json
   "scripts": {
   	"start": "nodemon --watch src/main --watch src/shared --exec './node_modules/.bin/electron' ./out/main/main.js"
   }
   ```

以后开发时只需要运行`npm start`就可做到主线程的自动刷新



### 打包主线程

主线程的开发过程我们可能会使用其他的构建工具, 这里我们同渲染线程一样, 将主线程的打包文件放在`out`目录中, 至此打包目录的结构应当如下

```
out
├── main - 主线程打包文件位置
│   └── main.js - 入口文件
├── renderer - 渲染线程打包位置
│   ├── .... 
│   └── index.html - 入口页面
└── shared - 公用文件
    └── utils.js
```



## 打包和发布

[electron-builder](https://github.com/electron-userland/electron-builder) 可以将我们的程序打包成可执行文件, 它的配置信息发在`package.json`中

这里配置的是`Mac`的打包信息, 具体的可以自行查阅文档

```json
{
  "main": "out/main/main.js", // 入口文件
  "scripts": {
	...
    "pack": "electron-builder -m --dir", // 简单打包软件, 用于测试
    "dist": "electron-builder -m",	     // 正式打包软件
    "build": "npm run build:renderer && npm run build:main && npm run dist" // 打包软件
  },
  "build": {
    "appId": "com.lleohao.sample",		 // 自行修改 
    "mac": {
      "category": "public.app-category.productivity" // 自行修改
    }
  }
}
```

运行`npm build`即可打包软件



## 开发流程

1. 运行`npm run dev:renderer`启动渲染线程开发
2. 运行`npm run dev:main`启动主线程开发
3. 运行`npm start`打开`Electron`程序
4. 运行`npm build`打包程序



示例项目地址 https://github.com/lleohao/electron-base
