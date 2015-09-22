# Understanding the Light Table Source Code

Light Table is built on top of Electron; some relevant docs:

 - [Electron Quick Start](https://github.com/atom/electron/blob/master/docs/tutorial/quick-start.md)
 - [Light Table Electron Guide](https://github.com/LightTable/LightTable/blob/master/doc/electron-guide.md)

An Electron app consists of a **main process** and some number of **renderer processes**. The main process is responsible for creating `BrowserWindow` instances for each web page to be displayed and each web page runs in its own renderer process.

From the Electron Quick Start doc:

> In web pages, calling native GUI related APIs is not allowed because managing native GUI resources in web pages is very dangerous and it is easy to leak resources. If you want to perform GUI operations in a web page, the renderer process of the web page must communicate with the main process to request that the main process perform those operations.
> 
> In Electron, we have provided the [ipc](https://github.com/atom/electron/blob/master/docs/api/ipc-renderer.md) module for communication between the main process and renderer process. There is also a [remote](https://github.com/atom/electron/blob/master/docs/api/remote.md) module for RPC style communication.

## *package.json*

The *package.json* file of an Electron app specifies the startup script of the app. Light Table's *deploy/core/package.json* file:

```
{
  "name": "LightTable",
  "description": "LightTable Electron App",
  "version": "0.8.0",
  "main": "main.js",
  "browserWindowOptions": {
    "icon": "img/lticon.png",
    "width": 1024,
    "height": 700,
    "min-width": 400,
    "min-height": 400,
    "web-preferences": {
      "webgl": true,
      "webaudio": true,
      "plugins": true
    }
  },
  "dependencies": {
    "codemirror": "^4.7.0",
    "optimist": "^0.5.2",
    "request": "^2.14.0",
    "wrench": "^1.5.6"
  },
  "forkedDependencies": {
    "replace": "^0.2.7",
    "socket.io": "^0.9.13",
    "shelljs": "^0.1.4",
    "tar": "^0.1.19"
  }
}
```

## *main.js*

Light Table's *deploy/core/main.js* file [rearranged so that the code flow is more linear]:

```
"use strict";

var app = require('app'),  // Module to control application life.
    BrowserWindow = require('browser-window'),  // Module to create native browser window.
    ipc = require("ipc"),
    optimist = require('optimist');

// Keep a global reference of the window object, if you don't, the window will
// be closed automatically when the javascript object is GCed.
var windows = {};
global.browserOpenFiles = []; // Track files for open-file event

var packageJSON = require(__dirname + '/package.json');

// Set $IPC_DEBUG to debug incoming and outgoing ipc messages for the main process
if (process.env["IPC_DEBUG"]) {
  var oldOn = ipc.on;
  ipc.on = function (channel, cb) {
    oldOn.call(ipc, channel, function() {
      console.log("\t\t\t\t\t->MAIN", channel, Array.prototype.slice.call(arguments).join(', '));
      cb.apply(null, arguments);
    });
  };
  var logSend = function (window) {
    var oldSend = window.webContents.send;
    window.webContents.send = function () {
      console.log("\t\t\t\t\tMAIN->", Array.prototype.slice.call(arguments).join(', '));
      oldSend.apply(window.webContents, arguments);
    };
  };
  var oldCreateWindow = createWindow;
  var createWindow = function() { logSend(oldCreateWindow()); };
}

start();

function start() {
  app.commandLine.appendSwitch('remote-debugging-port', '8315');
  app.commandLine.appendSwitch('js-flags', '--harmony');

  // This method will be called when electron has done everything
  // initialization and ready for creating browser windows.
  app.on('ready', onReady);

  // Quit when all windows are closed.
  app.on('window-all-closed', function() {
    app.quit();
  });

  // open-file operates in two modes - before and after startup.
  // On startup and before a window has opened, event paths are
  // saved and then opened once windows are available.
  // After startup, event paths are sent to available windows.
  app.on('open-file', function(event, path) {
    if (Object.keys(windows).length > 0) {
      Object.keys(windows).forEach(function(id) {
        windows[id].webContents.send('openFileAfterStartup', path);
      });
    }
    else {
      global.browserOpenFiles.push(path);
    }
  });
  parseArgs();
};

function onReady() {
  ipc.on("createWindow", function(event, info) {
    createWindow();
  });

  ipc.on("initWindow", function(event, id) {
    // Moving this to createWindow() causes js loading issues
    windows[id].on("focus", function() {
      windows[id].webContents.send("app", "focus");
    });
  });

  ipc.on("toggleDevTools", function(event, windowId) {
    if(windowId && windows[windowId]) {
      windows[windowId].toggleDevTools();
    }
  });

  createWindow();
};

function createWindow() {
  var browserWindowOptions = packageJSON.browserWindowOptions;
  browserWindowOptions.icon = __dirname + '/' + browserWindowOptions.icon;
  var window = new BrowserWindow(browserWindowOptions);
  windows[window.id] = window;
  window.focus();
  window.webContents.on("will-navigate", function(e) {
      e.preventDefault();
      window.webContents.send("app", "will-navigate");
  });

  if (process.platform == 'win32') {
    window.on("blur", function() {
      if (window.webContents)
        window.webContents.send("app", "blur");
    });
    window.on("focus", function() {
      if (window.webContents)
        window.webContents.send("app", "focus");
    });
  } else {
    window.on("blur", function() {
      window.webContents.send("app", "blur");
    });
    window.on("focus", function() {
      window.webContents.send("app", "focus");
    });
  }
  window.on("devtools-opened", function() {
    window.webContents.send("devtools", "disconnect");
  });
  window.on("devtools-closed", function() {
    window.webContents.send("devtools", "reconnect!");
  });

  // and load the index.html of the app.
  window.loadUrl('file://' + __dirname + '/LightTable.html?id=' + window.id);

  // Notify LT that the user requested to close the window/app
  window.on("close", function(evt) {
    window.webContents.send("app", "close!");
    evt.preventDefault();
  });

  // Emitted when the window is closed.
  window.on('closed', function() {
    windows[window.id] = null;
  });

  return window;
};

...
```

## *LightTable.html*

The `createWindow` function loads the Light Table 'home page' *deploy/core/LightTable.html*:

```
<html>
  <head>
    <meta charset='utf-8'>
    <title>
      Light Table
    </title>
    <link type="text/css" rel="stylesheet" href="css/reset.css"></link>
    <link type="text/css" rel="stylesheet" href="node_modules/codemirror/lib/codemirror.css"></link>
    <link type="text/css" rel="stylesheet" href="css/structure.css"></link>
    <style>
      html, body { background: #3b3f41; color:#ccc; height:100vh; width:100vw;}
      #loader { position:absolute; top:0; left:0; right:0; bottom:0; width:100vw; height:100vh; display:table; text-align:center; box-sizing:border-box; vertical-align:middle;}
      #loader p { display:table-cell; vertical-align:middle; text-align:center; }
    </style>
  </head>
  <body tabindex="-1">
    <div id="loader">
        <p>
            <img src="img/lighttabletextdark.png" width="40%"/>
        </p>
    </div>
    <div id="wrapper" style="opacity:0;">
    </div>
    <script src="node_modules/lighttable/util/keyevents.js" type="text/javascript"> </script>
    <script src="node_modules/lighttable/util/throttle.js" type="text/javascript"> </script>
    <script type="text/javascript">
        var script = document.createElement("script");
        script.type = "text/javascript";
        script.async = false;
        script.src= "node_modules/lighttable/bootstrap.js";
        document.body.appendChild(script);
        script.onload = function() {
            try {
                lt.objs.app.init();
            } catch (e) {
                console.error(e);
                console.log(e.stack);
            }
        }
    </script>
</body>
</html>
```

The final script block creates a new `script` block and loads the file *node_modules/lighttable/bootstrap.js* into it. It also runs the function `lt.objs.app.init()` when the script has finished loading. The *bootstrap.js* file is the build output of the core Light Table ClojureScript project.
