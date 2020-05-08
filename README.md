> this tutorial shows you how to make `pixi.js` game with `webpack` and `es6+`, how to optimize `image` assets and how to config `babel` to optimize `js` and convert`ES6+`to `ES5`.

[中文文档](./doc/README_ZH-CN.md)

### required 
* you should have `nodejs` installed and have some basic knowledge of `package.json` file, `npm init/install/run` command.
* you should have `google chrome` installed for running the project.
* basic knowledge of `webpack` and `babel`.
* it's better for you to have basic knowledge of `git`, the project [pixi-webpack-demo](https://github.com/proudcat/pixi-webpack-demo) is hosted on github, you can learn step by step by `git checkout` different branch, now `git clone` the project。

* for easy understanding, paste out the project directory here at the very first.

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

### [step1] initialize the project
> run command `git checkout init` to checkout `init` branch to learn this step.

* make directory `pixi-webpack-demo`, run `npm init` command under `pixi-webpack-demo` directory to initialize the project, fill the information, it will create a `package.json` file finally.
* run `npm install --save pixi.js` command to install `pixi.js`.
* after finish the two step before the `package.json` file should like this:

  ```json
  {
    "name": "pixi-webpack-demo",
    "version": "1.0.0",
    "description": "make pixi.js game with webpack",
    "main": "src/js/main.js",
    "keywords": ["pixi.js","webpack"],
    "author": "proudcat",
    "license": "MIT",
    "dependencies": {
      "pixi.js": "^5.2.1"
    }
  }
  ```
  
* create `src/index.html`。

  ```html
  <html>
    <head>
      <title>pixi-webpack-demo</title>
    </head>
    <body>
    <canvas id="scene"></canvas>
     <!-- we dont need import the entry js file here, webpack will help us to import the file automatically -->
     <!-- we will explain more at "[step2] import webpack" -->
    </body>
  </html>
  ```
* create entry js file `src/js/main.js`.

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

### [step2] import webpack
> run `git checkout webpack` command to checkout`webpack` branch to learn this step。
* run `npm install --save-dev webpack webpack-dev-server webpack-cli webpack-merge copy-webpack-plugin imagemin-webpack-plugin html-webpack-plugin` command to install the dependencies of webpack.

* create `webpack.common.js` file, this file is webpack common configuration.

  ```javascript
  const path = require('path')
  const HtmlPlugin = require('html-webpack-plugin')
  const CopyWebpackPlugin = require('copy-webpack-plugin')
  const ImageminPlugin = require('imagemin-webpack-plugin').default
  module.exports = {

    //context directory is src
    context: path.join(__dirname, 'src'),
    
    //entry file of the project,(relative to context)
    entry: ['./js/main.js'],
    output: {

      //distribution directory
      path: path.resolve(__dirname, 'dist'),

      /**
       * webpack will import the file for the index.html automatically,though the js file does not exist on disk.
       * the js file will generated after webpack build the project, and the js will inserted at index.html automatically.
       * [hash:8] means unique 8 digit hash generated everytime.
       **/
      filename: 'game.min.[hash:8].js',
    },
    target: 'web',
  
  plugins: [

    //copy all src/assets to dist/assets
    new CopyWebpackPlugin([
      { from: 'assets/',to:'assets/'}
    ], {
      ignore: [],
      debug:'debug',
      copyUnmodified: true
    }),

    //opimize all image file
    new ImageminPlugin({
      test: /\.(jpe?g|png|gif|svg)$/i ,

      // optipng: {
      //   optimizationLevel: 4
      // },

      //this way seems better on mac.
      pngquant: {
        verbose:true,
        quality: '80-90',
      }
    })

    //copy html to dist and insert the js reference.
    ,new HtmlPlugin({
      file:path.join(__dirname,'dist','index.html'),
      template:'./index.html'
    })
  ]
  }
  ```

* create `webpack.dev.js` file, the file used for debug phase.

  ```javascript
  const path = require('path')
  const merge = require('webpack-merge')
  const common = require('./webpack.common.js')
  module.exports = merge(common, {
    devtool: 'inline-source-map',
    mode: 'none',
    devServer: {

      //source code directory.
      contentBase: path.join(__dirname, 'src'),
      port: 8080,

      //if host set to 127.0.0.1, you cannot access the server on local network.
      host: '0.0.0.0',
      hot: true
    }
  })
  ```

* create `webpack.prod.js` file, the file is used for publishing the project(we will explain more detail at `[step3] import babel`).

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
* create start command at `script`section in `package.json` file.

  ```json
  {
    "name": "pixi-webpack-demo",
    "version": "1.0.0",
    "description": "make pixi.js game with webpack",
    "main": "src/js/main.js",
    "keywords": ["pixi.js","webpack"],
    "author": "proudcat",
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
* now we have import the `webpack` successfully, run `npm start` to start the project, it will open google chrome browser automatically,the game is running！try to modify the `src/js/main.js` file and save,the page will refresh automatically and we can see the modification has taken effect！

### [step3] import babel and publish the project

> run `git checkout master` to checkout `master` branch to learn this final step.

* with `babel`, you can write `ES6+` code（these lib is used fo converting ES6+ to ES5, and some pollyfill etc. look `babel` document for more details）。

  * `npm install --save-dev @babel/core @babel/plugin-transform-runtime @babel/preset-env babel-loader`
  * `npm install --save core-js @babel/runtime`

* install `rimraf` to clean the publish directory `dist`. run `npm install --save-dev rimraf` command to install it.

* create build command at `script` section in `package.json` file.

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
    "author": "proudcat",
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
* congratulations! run `npm run build` command, you can build the project now! you will find out the project will be published in `dist` directory,`js`has combined and optimized, `es6+` has converted to `es5`. all image file has been optimized. 
