### library项目

#### 项目结构

![image-20200903104041448](/Users/passerby/Library/Application%20Support/typora-user-images/image-20200903104041448.png)

#### 项目代码

```js
// main.js
const shortid = require("shortid");
const demo = require("./show.js");
module.exports = {
  shortid:function(){
   return shortid.generate();
  },
  getAuthorInfo:function(){
  	return "author name : gpasserby"
  },
  printinfo:function(){
   demo();
  }
};

// package.json
{
  "dependencies": {
    "shortid": "^2.2.8",
    "webpack": "^4.44.1"
  },
  "scripts": {
    "build": "webpack"
  },
  "name": "gpasserby-demo",
  "version": "1.0.12",
  "main": "bundle.js",
  "license": "MIT",
  "devDependencies": {
    "webpack-cli": "^3.3.12"
  }
}

// show.js
function demo(content){
window.document.getElementById("app").innerText="demo"+content;
}
module.exports=demo;

// webpack.config.js
const path = require("path");
module.exports = {
  entry:"./main.js",
  output:{
   filename:"bundle.js",
   path:path.resolve(__dirname,"./"),
   library:"passerby",
   libraryTarget:"commonjs2"
  }
}
```

#### 项目构建并发布

```bash
yarn && yarn run build && npm publish
```

### 测试项目

#### 初始化项目

```bash
npm init

npm install file:../demo
```







```js
{
  "dependencies": {
    "babel-loader": "^8.1.0",
    "shortid": "^2.2.8",
    "webpack": "^4.44.1"
  },
  "scripts": {
    "build": "webpack"
  },
  "name": "passerbyDate",
  "version": "1.0.0",
  "main": "bundle.js",
  "license": "MIT",
  "devDependencies": {
    "@babel/core": "^7.11.6",
    "@babel/plugin-transform-runtime": "^7.11.5",
    "@babel/polyfill": "^7.11.5",
    "@babel/preset-env": "^7.11.5",
    "webpack-cli": "^3.3.12"
  }
}


const path = require("path");
module.exports = {
  entry:"./main.js",
  output:{
   filename:"bundle.js",
   path:path.resolve(__dirname,"./"),
   library:"passerbyDate",
   libraryTarget:"umd",
   libraryExport: "default"
  },
  module: {
		rules: [
      {
        test: /(\.jsx|\.js)$/,
        loader: 'babel-loader',
        exclude: /(node_modules|bower_components)/
    	}
    ]
	},
	resolve: {
	    modules: [path.resolve('./node_modules'), path.resolve('./src')],
	    extensions: ['.json', '.js']
	}
}

```



小雅，生日快乐哈。<br />

算起来，咱俩认识五个多月了呢。<br />

从当初的吃鸡好友，到朋友，到恋人，到面基，到。。。现在。<br />

当初真的没有想到过，我们之间会发生这么多事，互相了解得这么深。<br />

也没想到，咱俩没能一直走下去，就这么结束了。<br />

虽然我们之间闹过了好多矛盾，到最后分手，<br />

但小雅，你真的是一个很优秀，很好的一个女孩儿。<br />

遇见你，跟你谈恋爱，是我最庆幸的事。<br />

你带给我很多快乐，很多温暖，很多幸福，也有一些不愉快。<br />

你问我，有没有喜欢过你？我真的很喜欢很喜欢你，你就住在我的心里。<br />

跟你相处，真的很开心，很幸福。你不要觉得自己不好。<br />

只是我不够好，期望你会遇到更好的人。<br />

不知道，你能不能看到我的这些话，这是我为了你生日而准备的，<br />

当初想说的话，并不是这些，只是现在。。。。<br />

你要过得好好的喔！小雅，生日快乐！！！<br />