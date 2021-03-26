# Webpack

> 将项目的js模块化代码、css、图片、高级es6转es5、TypeScript转es5、less转css、.vue转js等等打包成最基础的浏览器识别的文件

- `npm`淘宝镜像

  ```
  npm install -g cnpm --registry=https://registry.npm.taobao.org   //之后使用cnpm命令
  ```

- 安装`webpack`

  ```json
  //——全局安装
  npm install webpack[@版本] -g
  
  //——局部安装
  //cd对应目录
  npm install webpack[@版本] --save-dev//--save-dev为开发时依赖，项目打包后不需要继续使用
  ```

- 模块打包

  ```
  webpack ./src/main.js ./dist/bundle.js    
  //报错输入:set-ExecutionPolicy RemoteSigned 
  ```

- 生成`package.json`

   ```json
    npm init -y//-y表示配置信息都是默认
   ```
## 配置

- `webpack.config.js`（固定名字，根据配置文件打包）

  ```javascript
  const path = require('path');
  
  module.exports = {
    entry: './path/to/my/entry/file.js',//入口
    output: {
      path: path.resolve(__dirname, 'dist'),//出口文件绝对路径
      filename: 'my-first-webpack.bundle.js'//出口文件名
    }
  };
  ```

- 打包命令

  ```
  webpack  
  ```

  一般我们不用上述命令(上述命令使用的是全局webpack版本，一般项目依赖特定的webpack版本），因此需要配置`package.json`

  ```json
  {
    "name": "_1",
    "version": "1.0.0", 
    "description": "",
    "main": "webpack.config.js",
    "scripts": {
      "test": "echo \"Error: no test specified\" && exit 1",
      "build":"webpack"//加上这一句，优先使用局部webpack的版本来打包（本地项目node_modules/.bin下）
    },
    "keywords": [],
    "author": "",
    "license": "ISC"
  }
  ```

  再将webpack命令替换为

  ```json
  npm run build  //等价于webpack命令
  ```

  

## Loader

  > webpack 只能理解 JavaScript 和 JSON 文件。**loader** 让 webpack 能够去处理其他类型的文件，并将它们转换为有效 [模块](https://v4.webpack.docschina.org/concepts/modules)，以供应用程序使用，以及被添加到依赖图中。

  **CSS、图片需要的loader**

  `webpack.config.js`下

  ```javascript
  const path = require('path');
  
  module.exports = {
    entry: './src/main.js',
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: 'bundle.js'
    },
    module: {
      rules: [
        {
          test: /\.css$/,//css
          use: ['style-loader', 'css-loader'],
        },
        {
          test: /\.(png|jpg|gif)$/i,//图片
          use: [
            {
              loader: 'url-loader',
              options: {
                limit: 8192//小于8kb图片以base64格式加载，超过该限制则还需要file-loader
              }
            }
          ]
        }
      ],
    }
  };
  ```

  

  ## npm run build/npm run dev

<img src="pic\image-20210325150913794.png" alt="image-20210325150913794" style="zoom:67%;" />

<img src="pic\image-20210325150948444.png" alt="image-20210325150948444" style="zoom:67%;" />

  

  

  

  



 

