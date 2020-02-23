> 本文介绍怎么使用`webpack`搭建`pixi.js`游戏的开发环境，怎么配置`babel`优化代码(tree shake)、混淆代码，并最终将`ES6+`转换为`ES5`,怎么优化`图片`资源并最终发布项目。

### 前提
* 需要会简单使用`nodejs`，了解`package.json`，会简单使用`npm init`，`npm install`，`npm run`命令。

* 需要稍微了解`webpack`和`babel`。

* 需要有`google chrome`浏览器。

* 最好会一点`git`，demo项目[pixi-webpack-demo](https://github.com/proudcat/pixi-webpack-demo)托管在`github`上，通过切换不同分支演示一步一步项目的构建过程，现在把项目`clone`下来吧。

* 为了更容易理解，这里先贴出来项目最终的目录结构

  ```tex
  .
  ├── dist
  │   ├── index.html
  │   ├── game.min.879458fc.js
  │   └── assets
  │       └── bunny.png
  ├── src
  │   ├── index.html
  │   ├── assets
  │   │   └── bunny.png
  │   └── js
  │       └── main.js
  ├── package.json
  ├── webpack.common.js
  ├── webpack.dev.js
  └── webpack.prod.js
  ```

  

### 构建环境
* `nodejs`：需要node环境，前端项目现在基本都是基于node项目创建的，node的包管理系统和工具链很方便。
* `git`：非必须，看demo时候切分支用。

### 初始化项目
> 运行`git checkout init`切换到`init`分支即可看到这一步的示例。
* 创建目录`pixi-webpack-demo`，在`pixi-webpack-demo`根目录下运行`npm init`命令初始化项目，按照提示输入项目信息，完成后生成一个`package.json`文件。
* 运行`npm install --save pixi.js`安装依赖。
* 完成上面两步，`package.json`文件如下所示：

  ```json
  {
    "name": "pixi-webpack-demo",
    "version": "1.0.0",
    "description": "make pixi.js game with webpack",
    "main": "src/js/main.js",
    "keywords": ["pixi.js","webpack"],
    "author": "yulijun",
    "license": "MIT",
    "dependencies": {
      "pixi.js": "^5.2.1"
    }
  }
  ```
  
* 创建文件`src/index.html`。

  ```html
  <html>
    <head>
      <title>pixi-webpack-demo</title>
    </head>
    <body>
    <canvas id="scene"></canvas>
     <!--这里不需要引入入口js，webpack会自动帮我们引入，“引入webpack”步骤再详细解释它-->
    </body>
  </html>
  ```
* 创建文件`src/js/main.js`，这个文件是游戏入口文件。

  ```javascript
  import * as PIXI from 'pixi.js'
  
  const app = new PIXI.Application({
    width: 720,
    height: 1280,
    backgroundColor: 0x1099bb,
    view: document.querySelector('#scene')
  });
  
  const texture = PIXI.Texture.from('assets/bunny.png');
  const bunny = new PIXI.Sprite(texture);
  bunny.anchor.set(0.5);
  bunny.x = 160
  bunny.y = 160
  app.stage.addChild(bunny);
  
  app.ticker.add((delta) => {
    bunny.rotation -= 0.01 * delta;
  });
  ```

### 引入webpack
> 运行`git checkout webpack`切换到`webpack`分支即可看到这一步的示例。
* 运行`npm install --save-dev webpack webpack-dev-server webpack-cli webpack-merge copy-webpack-plugin imagemin-webpack-plugin html-webpack-plugin`安装依赖。

* 创建`webpack.common.js`文件，这个是webpack公共配置。

  ```javascript
  const path = require('path')
  const HtmlPlugin = require('html-webpack-plugin')
  const CopyWebpackPlugin = require('copy-webpack-plugin')
  const ImageminPlugin = require('imagemin-webpack-plugin').default
  module.exports = {
    context: path.join(__dirname, 'src'),
    
    //游戏入口文件
    entry: ['./js/main.js'],
    
    output: {
      //js文件最终发布到哪个路径
      path: path.resolve(__dirname, 'dist'),
      /**
       * 开发调试阶段webpack会自动处理这个文件让html引用到，虽然磁盘上不会有这个文件。
       * 但是最终发布项目的时候会生成这个文件，并会插入到index.html中。
       * [hash:8]的意思是生成随机的八位hash值，为了缓存更新问题。
       **/
      filename: 'game.min.[hash:8].js',
    },
    target: 'web',
  
  plugins: [

    //拷贝图片资源
    new CopyWebpackPlugin([
      { from: 'assets/',to:'assets/'}
    ], {
      ignore: [],
      debug:'debug',
      copyUnmodified: true
    }),

    //压缩图片资源
    new ImageminPlugin({
      test: /\.(jpe?g|png|gif|svg)$/i ,

      //这种方式压缩在mac上效果不太好。
      // optipng: {
      //   optimizationLevel: 4
      // },

      //这个方式在mac上压缩效果更好,windows上尚未测试有待验证。
      pngquant: {
        verbose:true,
        quality: '80-90',
      }
    }),

    //拷贝html，插入js。
    new HtmlPlugin({
      file:path.join(__dirname,'dist','index.html'),
      template:'./index.html'
    })
  ]
  }
  ```

* 创建`webpack.dev.js`文件，这个配置文件用于开发调试阶段。

  ```javascript
  const path = require('path')
  const merge = require('webpack-merge')
  const common = require('./webpack.common.js')
  module.exports = merge(common, {
    devtool: 'inline-source-map',
    mode: 'none',
    devServer: {
      //调试时候源代码的位置
      contentBase: path.join(__dirname, 'src'),
      port: 8080,
      host: '0.0.0.0',
      hot: true
    }
  })
  ```

* 创建`webpack.prod.js`文件，这个配置文件用于发布项目(稍后在`引入babel`和`发布项目`步骤再详细介绍，这里暂时先贴出来)，这里配置了`babel`转码、`tree shake`和生成`source map`等。

  ```javascript
  const merge = require('webpack-merge')
  const common = require('./webpack.common.js')
  module.exports = merge(common, {
    'mode':'production',
    devtool: 'source-map',
    module: {
      rules: [{
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', {
                'corejs': '3',
                'useBuiltIns': 'usage'
              }]
            ],
            plugins: ['@babel/plugin-transform-runtime']
          }
        }
      }]
    }
  })
  ```
* 在`package.json`中的`script`配置节增加启动命令。

  ```json
  {
    "name": "pixi-webpack-demo",
    "version": "1.0.0",
    "description": "make pixi.js game with webpack",
    "main": "src/js/main.js",
    "keywords": ["pixi.js","webpack"],
    "author": "yulijun",
    "license": "MIT",
    "scripts": {
      "start": "webpack-dev-server --open 'google chrome' --config webpack.dev.js"
    },
    "devDependencies": {
      "webpack": "^4.41.5",
      "webpack-cli": "^3.3.10",
      "webpack-dev-server": "^3.10.3",
      "copy-webpack-plugin": "^5.1.1",
      "html-webpack-plugin": "^3.2.0",
      "imagemin-webpack-plugin": "^2.4.2",
      "webpack-merge": "^4.2.2"
    },
    "dependencies": {
      "pixi.js": "^5.2.1"
    }
  }
  ```
* 现已成功引入了`webpack`，运行`npm start`启动项目，会自动打开chrome浏览器，我们看到游戏已经跑起来了！尝试修改`src/js/main.js`文件，保存下，页面会自动刷新，我们的修改也已经能反映到页面上了！

### 构建项目

> 运行`git checkout master`切换到`master`分支即可看到这最终一步的示例。

* 引入`babel`让你能使用最新的ES特性（这些库主要是为了ES6+转ES5，还有些pollyfill等等，这里不做过多的解释，具体可参考`babel`官方文档）。

  * `npm install --save-dev @babel/core @babel/plugin-transform-runtime @babel/preset-env babel-loader`
  * `npm install --save core-js @babel/runtime`

* 引入`rimraf`，用于在发布前删除发布目录`dist`。运行`npm install --save-dev rimraf`安装依赖。

* 在`package.json`中`script`节加入构建相关命令，然后`run npm build`就能成功打包了！

  ```json
  {
    "name": "pixi-webpack-demo",
    "version": "1.0.0",
    "description": "make pixi.js game with webpack",
    "main": "src/js/main.js",
    "scripts": {
      "start": "webpack-dev-server --open 'google chrome' --config webpack.dev.js",
      "clean": "rimraf dist",
      "prebuild": "npm run clean",
      "build": "webpack --config webpack.prod.js"
    },
    "author": "yulijun",
    "keywords": ["pixi.js","webpack","pixijs","web","game"],
    "license": "MIT",
    "devDependencies": {
      "@babel/core": "^7.8.4",
      "@babel/plugin-transform-runtime": "^7.8.3",
      "@babel/preset-env": "^7.8.4",
      "babel-loader": "^8.0.6",
      "rimraf": "^3.0.2",
      "webpack": "^4.41.5",
      "webpack-cli": "^3.3.10",
      "webpack-dev-server": "^3.10.3",
      "webpack-dev-server": "^3.10.3",
      "copy-webpack-plugin": "^5.1.1",
      "html-webpack-plugin": "^3.2.0",
      "webpack-merge": "^4.2.2"
    },
    "dependencies": {
      "@babel/runtime": "^7.8.4",
      "core-js": "^3.6.4",
      "pixi.js": "^5.2.1"
    }
  }
  ```

* 恭喜你，至此开发和构建环境已经全部完成，可尝试在源码中添加一些`es6+`语法，然后运行`npm run build`构建项目，最终打包好的项目会在`dist`目录中，`js`已经被混淆并合并，无用的引用通过`tree shake`已经被去掉了，包尺寸优化到了最小，而且所有`es6+`的语法均转换为`es5`以适应更多的浏览器。所有的图片也都进行了压缩处理。