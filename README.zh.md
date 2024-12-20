# ulanzideck-plugin-sdk node版本


<p align="start">
   <a href="./README.md">English</a> | <strong>简体中文</strong>
</p>

## 简介
我们依据插件开发协议，封装了与上位机的WebSocket连接及相关的通信事件。这样简化了开发流程，使开发者仅需通过简单的事件调用即可实现与上位机的通信，从而能更专注于插件功能的开发。


```bash
当前版本根据 Ulanzi JS 插件开发协议-V1.2.2 来编写
```


## 文件介绍
```bash
plugin-common-node   //通用node包
├── libs
│   ├── constants.js      //上位机的事件常量，无需二次编写
│   ├── randomPort.js      //生成随机端口
│   ├── utils.js          //一些常用方法的封装
│   └── ulanzideckApi.js    //包括 ulanzi所有事件的封装，socket的连接
├── index.js  //入口文件
```


## 使用

### 一些说明和约定

1. 插件库的主服务（例app.js）会一直与上位机连接，用于做主要功能，包括上位机icon的更新等。

2. 插件库的配置项（例inspector.html），配置项我们后续称为action。切换功能按键之后就会被销毁，不宜做功能处理。主要用于发送配置项到上位机和同步上位机数据。

3. 为了统一管理，我们的插件包的名称为 com.ulanzi.插件名.ulanziPlugin

4. 为了通用库的正常使用，主服务连接的uuid我们约定长度是4。例：com.ulanzi.ulanzideck.插件名

5. 配置项连接的uuid要大于4用于区分。例：com.ulanzi.ulanzideck.插件名.插件action

6. 在使用node来做主服务时，为了避免端口冲突，请通过 plugin-common-node 提供的 RandomPort 来生成端口。具体可查看[<a href="#title-2">2. 生成随机端口</a>]

7. 由于本地的node环境和上位机node环境运行的区别，以及程序打包之后获取本地路径会出现一些的bug，因此我们提供Utils.getPluginPath()方法来获取插件根目录的本地路径，大家按需使用。具体可查看[<a href="#title-3">3. 获取插件根目录的路径</a>]

### 使用步骤

```bash
SDK(node版本)的具体使用和文件夹规范，可以查看 demo/com.ulanzi.teamspeak5.ulanziPlugin 的实现。
以下简单介绍通用库的使用：
```
#### * 特殊参数 context
由于一个action功能会配置到多个按键key上，因此common库为大家拼接了一个唯一值context。在我们创建功能实例的时候，只需保存唯一值context。若需要更新数据时，再根据对应的唯一值context，即可将消息发送到对应的key值上。
```bash
1. 特殊参数context, 是common库拼接出的唯一值，它连同接收到的message一起传递给主服务和action。

2. context的拼接规则是 uuid + '___' + key + '___' + actionid，由对应的$UD.encodeContext(msg)生成。

3. 同时我们提供 $UD.decodeContext(context) 来解构唯一值，返回 { uuid, key, actionid }。

4. 由于clear事件的param是数组形式，因此clear的context拼接在param里。请大家做clear处理时，注意循环获取。

```

#### 1. 下载引用

1. 将plugin-common-node下载到本地，并将文件夹复制到运行目录下。
2. plugin-common-node 基于 ws ，因此需要在项目根目录中安装ws依赖包。
3. 然后根据文件夹的位置来引用即可。


#### <span id="title-2">2. 生成随机端口</span>
调用随机生成接口的方法getPort()之后，将在插件主服务下会自动生成ws-port.js ，该js文件内容为 window.__port = 端口号;
action的html可以通过引入'ws-port.js"文件，获得主服务的端口，连接到node主服务


```js
import { RandomPort } from './actions/plugin-common-node/index.js';

const generatePort = new RandomPort(); 

//生成随机端口
const port = generatePort.getPort(); 

```


#### <span id="title-3">3. 获取插件根目录的路径</span>

Utils.getPluginPath()方法可以获取插件根目录的本地路径，兼容Windows和Mac系统，大家按需使用。

```js
import { Utils } from './actions/plugin-common-node/index.js';

//获取根目录文件路径
const _pluginPath = Utils.getPluginPath()

console.log('Plugin path: ', _pluginPath)

```

#### 4. 连接上位机
以下简单展示一些方法的使用，具体可查看[<a href="#title-5">5. 接收事件 上位机->插件</a>][<a href="#title-6">6. 发送事件 插件->上位机</a>]
```js
  import { UlanzideckApi } from './actions/plugin-common-node/index.js';;

  const $UD = new UlanzideckApi();
  //连接socket，连接成功后，会触发事件onConnected
  $UD.connect('com.ulanzi.ulanzideck.teamspeak5');

  $UD.onConnected(conn => {
    //表示已连接
  })

  //接收action拖入键盘的事件
  $UD.onAdd( message => {
    //保存action实例
  })

  //接收action初始化参数
  $UD.onParamFromApp( message => {
      //保存action初始化参数
  })


  //插件清除
  $UD.onClear( message => {
     //实现功能，清除插件
    if(message.param){
      for(let i = 0; i<message.param.length; i++){
        const context = message.param[i].context
        console.log('===context clear', context)

      }
    }
  })

  //配置icon
  function serIcon(context, data, text){
    $UD.setBaseDataIcon(context, data, text) 
  }


```

#### <a id="title-5">5. 接收事件 上位机->插件</a>
```js
/**
 * 监听websocket连接事件，以及上位机发出的事件
*/
1. $UD.onConnected(conn => ())  //websocket连接成功
2. $UD.onClose(conn => ())  // websocket 断开连接
3. $UD.onError(conn => ())  //websocket 错误
4. $UD.onAdd(message => ())     //接收上位机发出 "cmd": "add" 的事件
5. $UD.onParamFromApp(message => ())  //接收上位机发出 "cmd": "paramfromapp" 的事件
6. $UD.onParamFromPlugin(message => ())  //接收上位机发出 "cmd": "paramfromplugin" 的事件
7. $UD.onRun(message => ())  //接收上位机发出 "cmd": "run" 的事件
8. $UD.onSetActive(message => ())  //接收上位机发出 "cmd": "setactive" 的事件
9. $UD.onClear(message => ())  //接收上位机发出 "cmd": "clear" 的事件
10. $UD.onSelectdialog(message => ())  //接收上位机返回的 "cmd": "selectdialog" 的事件，用于接收选择文件/文件夹的结果


```

#### <a id="title-6">6. 发送事件 插件->上位机</a>

```js
/**
 * 向上位机发送配置参数
 * @param {object} settings 必传 | 配置参数
 * @param {object} context 可选 | 唯一值。非必传，由action页面发出时可以不传，由主服务发出必传
*/
1. $UD.sendParamFromPlugin(settings, context) 

/**
 * 设置图标-使⽤配置⾥的图标列表编号，请对照manifest.json。
 * @param {string} context 必传 |唯一值, 接收到的message里面common库会自动拼接给出
 * @param {number} state 必传 | 图标列表编号，
 * @param {string} text 可选 | icon是否显示文字
*/
2. $UD.setStateIcon(context, state, text) 


  /**
 * 设置图标-使⽤⾃定义图标
 * @param {string} context 必传 |唯一值,每个message里面common库会自动拼接给出
 * @param {string} data 必传 | base64格式的icon
 * @param {string} text 可选 | icon是否显示文字
*/
3. $UD.setBaseDataIcon(context, data, text) 


/**
 * 设置图标-使⽤本地图片文件
 * @param {string} context 必传 |唯一值,每个message里面common库会自动拼接给出
 * @param {string} path  必传 | 本地图片路径，⽀持打开插件根⽬录下的url链接（以/ ./ 起始的链接）
 * @param {string} text 可选 | icon是否显示文字
*/
4. $UD.setPathIcon(context, path, text) 


/**
 * 设置图标-使⽤⾃定义的动图
 * @param {string} context 必传 |唯一值,每个message里面common库会自动拼接给出
 * @param {string} gifdata  必传 | ⾃定义gif的base64编码数据
 * @param {string} text 可选 | icon是否显示文字
*/
5. $UD.setGifDataIcon(context, gifdata, text) 



  /**
 * 设置图标-使⽤本地gif⽂件
 * @param {string} context 必传 |唯一值,每个message里面common库会自动拼接给出，
 * @param {string} gifdata  必传 | 本地gif图片路径，⽀持打开插件根⽬录下的url链接（以/ ./ 起始的链接）
 * @param {string} text 可选 | icon是否显示文字
*/
6. $UD.setGifPathIcon(context, gifpath, text) 


/**
 * 请求上位机弹出Toast消息提⽰
 *  @param {string} msg 必传 | 窗口级消息提示
*/
7. $UD.toast(msg) 

/**
 * 请求上位机弹出选择对话框:选择文件
 *  @param {string} filter 可选 | 文件过滤器。筛选文件的类型，例如 "filter": "image(*.jpg *.png *.gif)" 或者 筛选文件 file(*.txt *.json) 等
 * 该请求的选择结果请通过 onSelectdialog 事件接收
*/
8. $UD.selectFileDialog(filter) 


/**
 * 请求上位机弹出选择对话框:选择文件夹
 * 该请求的选择结果请通过 onSelectdialog 事件接收
*/
9. $UD.selectFolderDialog() 


/**
   * 请求上位机使⽤浏览器打开url
   * @param {string} url 必传 | 直接远程地址和本地地址，⽀持打开插件根⽬录下的url链接（以/ ./ 起始的链接）。
   *                            只能是基本路径，不能带参数，需要带参数请设置在param值里面
   * @param {local} boolean 可选 | 若为本地地址为true
   * @param {object} param 可选 | 路径的参数值
  */
10. $UD.openUrl(url, local, param)


/**
 * 请求上位机机显⽰弹窗；弹窗后，test.html需要主动关闭，测试到window.close()可以通知弹窗关闭
 *  @param {string} url 必传 | 本地html路径。只能是基本路径，不能带参数，需要带参数请设置在param值里面
 * @param {string} width 可选 | 窗口宽度，默认200
 * @param {string} height 可选 | 窗口高度，默认200
 * @param {string} x 可选 | 窗口x坐标，不传值默认居中
 * @param {string} y 可选 | 窗口y坐标，不传值默认居中
 * @param {object} param 可选 | 路径的参数值
*/
11. $UD.openView(url, width = 200, height = 200, x , y , param)


```