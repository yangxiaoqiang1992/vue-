# cocall5集成

小宇助手即时消息需要集成cocall5通讯工具，这里总结一下集成cocall5的注意点，此方案也适用于vue大型项目，适用于需要做模块分割，既满足集成部署，又能满足各模块单独部署场景

## 技术难点：
 cocall5后台采用java，api请求采用c++封装，前端使用electron-vue node es6，技术栈较为复杂，集成难度比较大

## 解决方案：
 思路：将cocall5代码vue部分全部复制到项目，不修改文件夹结构和路由，webpack里配置alias ,集成后涉及到很多依赖和文件路径错误，用别名可以方便的对引用路径做修改；配置多入口，解决cocall的路由与vuex与原项目冲突的麻烦，即各系统管理各自的路由和vuex

 详细步骤如下：
 * 新建一个cc文件夹，将cocall5 renderer文件夹下所有文件都复制过去
 * 将cocall5的`index.ejs`文件复制到cc文件夹下
 * webpack.render.config.js添加@cc别名，将复制后文件夹的路径修改为@cc下，添加别名需要重启项目
    ```
      resolve: {
        alias: {
          '@': path.join(__dirname, '../src/renderer'),
          'vue$': 'vue/dist/vue.esm.js',
          '@cc': path.join(__dirname, '../src/cc')
        },
        extensions: ['.js', '.vue', '.json', '.css', '.node']
      },
    ```
* 启动项目会报依赖和相关文件路径的报错，按照所需依赖npm install 安装相关版本依赖
* 配置electron-vue多入口
  webpack.renderer.config.js文件需要配置entry,与vue配置多入口略有不同，需要将多个路径添加到数组里，且`key` 值必须为`renderer`,数组的每一项都指向各项目的入口js
  ```
    entry: {
      renderer: [path.join(__dirname, '../src/renderer/main.js'), path.join(__dirname, '../src/cc/main.js')]
    },
  ```
  添加入口之后，需要为程序指定多入口和各自加载的模板，需要在`plugins`下添加入口模板,有几个入口，就添加几个 `new HtmlWebpackPlugin()`,修改`filename`和`template`，`template`指向各项目入口`index.ejs`
  ```
    plugins: [
    new VueLoaderPlugin(),
    new MiniCssExtractPlugin({filename: 'styles.css'}),
    new HtmlWebpackPlugin({
      filename: 'index.html',
      template: path.resolve(__dirname, '../src/index.ejs'),
      minify: {
        collapseWhitespace: true,
        removeAttributeQuotes: true,
        removeComments: true
      },
      nodeModules: path.resolve(__dirname, '../node_modules')
    }),
    new HtmlWebpackPlugin({
        filename: 'cc.html',
        template: path.resolve(__dirname, '../src/cc/index.ejs'),
        minify: {
           collapseWhitespace: true,
           removeAttributeQuotes: true,
           removeComments: true
      },
        nodeModules: path.resolve(__dirname, '../node_modules')
    }),
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoEmitOnErrorsPlugin()
  ],
  ```
  * vue多项目集成或大的项目需要做模块分割，既能分开部署，又能集成到一个大的项目，解决方法与此类似，但又不尽相同，参考下方vue-cli多页面配置

## electron-vue双击位置跑偏问题

electron默认双击会最大化或正常化窗口，而真实项目中，登录页是不需要双击最大化，所以可在生成窗口配置或不需要最大化的页面动态修改 `BrowserWindow` 的参数，禁用最大化或最小化窗口

```
  minimizable:false,
  maximizable:false,
```
##  新消息通知问题

小宇助手需要在cocall获取到新消息的时候图标闪烁，这一版直接使用主线程进行通信，cocall5有托盘 `tray.js`，托盘显示新消息，所以可利用托盘消息的方式来向小宇助手发送会话消息，搜索`tray-messages-update`,可在主线程监听 `tray-messages-update` 的时候将传到主线程的数据直接转发到小宇窗口；

```
  ipcMain.on('tray-messages-update', (event, data) => {
    this.trayWindow.webContents.send('tray-messages-update', data);
    ...在此添加向小宇发送获取未读消息代码
    // console.log(data)
    if(data.count > 0){
      this.startTrayBlink();
    } else {
      this.stopTrayBlink();
    }
  });
```

或在cocall的会话模块向主线程发送 `tray-messages-update` 消息时，添加另一个线程监听，专用于通知小宇助手` ipcRenderer.send('notify-xyzs-update', arg);`

```
 refreshChatUnreadCount (state) {
        let count = 0;
        let unReadSessions = state.chatSessions.filter(session => {
            if(session.unread > 0 && !session.muted){
                count += session.unread;
                return true;
            }
        });
        // 初始化时，显示未读消息数据
        state.chatUnreadCount = Object.assign(count);

        let arg = {
            count : count,
            sessions : unReadSessions
        }
        // 有未读消息，通知主进程进行操作
        ipcRenderer.send('tray-messages-update', arg);
        ipcRenderer.send('notify-xyzs-update', arg);
    },
```
##  集成打包问题

场景：多项目集成打包时，由于cocall有单独的build.js文件，直接使用cocall的打包配置，则无法打包其他项目的文件；而如果使用electron-vue默认的打包配置，则无法将cocall核心的c++编译之后的代码进行打包

解决方法：
如果有使用 `electron-builder.json` 配置文件进行打包，需要修改 `files` 字段数组，将位于项目根目录的cocall编译之后的 `addons` 文件夹和 `cc-module-wrapper`  `cc-update`  `Cocall-Core` 文件夹及文件加下所有子目录及文件，添加到打包目录;如果不使用 `electron-builder.json` ，也可直接在 `package.json` 直接写上配置，如下所示：

```
  "build": {
    "productName": "即知-智能助手",
    "appId": "com.thusnisoft.xiaoyu",
    "copyright": "北京华宇信息技术有限公司",
    "compression": "store",
    "directories": {
      "output": "build"
    },
    "asar": false,
    "files": [
      "dist/electron/**/*",
      "addons/**",
      "cc-module-wrapper/**",
      "cc-updater/**",
      "resources/**",
      "CoCall-Core/**"
    ],
    "nsis": {
      "artifactName": "${productName}-${os}-${arch}-${version}.${ext}",
      "oneClick": false,
      "perMachine": true,
      "allowToChangeInstallationDirectory": true,
      "installerIcon": "build/icons/xiaoyu.ico",
      "uninstallerIcon": "build/icons/xiaoyu.ico",
      "installerHeader": "",
      "installerHeaderIcon": "build/icons/xiaoyu.ico",
      "installerSidebar": "",
      "uninstallerSidebar": "",
      "menuCategory": "Thunisoft",
      "unicode": true,
      "installerLanguages": "zh_CN",
      "language": "2052"
    },
  },
```

参考链接
  * [vue配置多入口](https://www.jianshu.com/p/49124deb4e14)
  * [多入口与单页面](https://www.cnblogs.com/xiyangcai/p/8609773.html)

