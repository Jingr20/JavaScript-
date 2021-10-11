### 【Promise】

##### 1、将原生的ajax封装成promise

```javascript
// 返回promise对象
function ajax(url){
  return new Promise((resolve,reject) => {
    var xhr = new XMLHttpRequest();
    xhr.open('GET',url);
    xhr.send();
    xhr.onreadyStateChange = function(){
      // readyState状态改变时会触发该函数
      // 通过readyState === 4判断请求是否完成，如果已完成，再根据status === 200判断是否是一个成功的响应
      if(this.readyState === 4 && this.status === 200){
        // 以JSON格式返回响应数据
        resolve(JSON.parse(this.response));
      }else{
        reject('加载错误');
      }
    };
    xhr.onerror = ()=>{
      reject(this.error);
    };
  });
}
```

**补充知识点：**

- **readyState（状态值）和status（状态码）的区别**

  readyState，是指运行AJAX所经历过的几种状态，无论访问是否成功都将响应的步骤，可以理解成为AJAX运行步骤，使用“ajax.readyState”获得；

  status，是指无论AJAX访问是否成功，由HTTP协议根据所提交的信息，服务器所返回的HTTP头信息代码，使用“ajax.status”获得；

  总体理解：可以简单的理解为state代表一个整体的状态。而status是这个大的state下面具体的小的状态。

- **readyState的五种状态值——用来标识当前XMLHttpRequest对象处于什么状态**

  0 (未初始化)： (XMLHttpRequest)对象已经创建，但还没有调用open()方法

  1 (载入)：已经调用open() 方法，但尚未发送请求

  2 (载入完成)：已经调用了send()函数， 请求已经发送完成，但还未收到服务器回应

  3 (交互)：可以接收到部分响应数据（正在接受服务器返回的数据）

  4 (完成)：已经接收到了全部数据，并且连接已经关闭

  **另外一版本（推荐）：**

  0 (未初始化)： (XMLHttpRequest)对象已经创建，但还没有调用open()方法

  1 (载入/正在发送请求)：对XMLHttpRequest对象进行初始化，即调用open()方法，根据参数(method,url,true)，完成对象状态的设置。并调用send()方法开始向服务端发送请求。值为1表示正在向服务端发送请求。

  2 (载入完成/数据接收)：此阶段接收服务器端的响应数据。但获得的还只是服务端响应的原始数据，并不能直接在客户端使用。值为2表示send()方法执行完成，已经接收完全部响应数据。并为下一阶段对数据解析作好准备。

  3 (交互/解析数据)：正在解析响应内容

  4 (完成/数据解析完毕)：此阶段确认全部数据都已经解析为客户端可用的格式，解析已经完成。

  总之，整个XMLHttpRequest对象的生命周期应该包含如下阶段： 
  创建－0初始化请求－1发送请求－2接收数据－3解析数据－4完成 。

- **status——XMLHttpRequest对象的一个属性，表示响应的HTTP状态码**

  在HTTP1.1协议下，HTTP状态码总共可分为5大类

  ```javascript
  1xx：信息响应类，表示接收到请求并且继续处理
  2xx：处理成功响应类，表示动作被成功接收、理解和接受
  3xx：重定向响应类，为了完成指定的动作，必须接受进一步处理
  4xx：客户端错误，客户请求包含语法错误或者是不能正确执行
  5xx：服务端错误，服务器不能正确执行一个正确的请求
  ```

- 思考问题：为什么onreadystatechange的函数实现要同时判断readyState和status呢？

  只使用readyState:

  服务响应出错了，但还是返回了信息，这并不是我们想要的结果。由于只使用readystate做判断，它不理会返回的结果是200、404还是500，只要请求完成了，就执行接下来的javascript代码，结果将造成各种不可预料的错误。所以只使用readyState判断是行不通的。

  只使用status判断：

  不止是readyState为4时status等于200，前面情况下也会有。

##### 2、简单的实现一个Promise

https://github.com/forthealllight/blog/issues/4

- **明确Promise/A+规范**

  **Promise/A+术语**

  （1）"promise"是一个对象或者函数，该对象或者函数有一个then方法

  （2）"thenable"是一个对象或者函数，用来定义then方法

  （3）"value"是promise状态成功时的值

  （4）"reason"是promise状态失败时的值

  我们明确术语的目的，是为了在自己实现promise时，保持代码的规范性

  **Promise/A+要求**

  （1）Promise是一个类，在执行这个类时要传递一个执行器进去，这个执行器会立即执行；

  （2）一个promise必须有3个状态，pending，fulfilled(resolved)，rejected当处于pending状态的时候，可以转移到fulfilled(resolved)或者rejected状态。当处于fulfilled(resolved)状态或者rejected状态的时候，就不可变。

  resolve和reject函数是用来更改状态的

  （3）一个promise必须有一个then方法，then方法接受两个参数：

  ```
  promise.then(onFulfilled,onRejected)
  ```

  其中onFulfilled方法表示状态从pending——>fulfilled(resolved)时所执行的方法，参数是成功之后的值；而onRejected表示状态从pending——>rejected所执行的方法，参数是失败后的原因

  （4）为了实现链式调用，then方法必须返回一个promise

  ```
  promise2=promise1.then(onFulfilled,onRejected)
  ```

- **实现一个符合Promise/A+规范的Promise（最简版）**

  ```javascript
  // 参数constructor为函数
  function MyPromise(constructor){
      let self=this;
      self.status="pending" //定义状态改变前的初始状态
      self.value=undefined;//定义状态为resolved的时候的值
      self.reason=undefined;//定义状态为rejected的时候的值
      function resolve(value){
          //两个==="pending"，保证了状态的改变是不可逆的
         if(self.status==="pending"){
            self.value=value;
            self.status="resolved";
         }
      }
      function reject(reason){
          //两个==="pending"，保证了状态的改变是不可逆的
         if(self.status==="pending"){
            self.reason=reason;
            self.status="rejected";
         }
      }
      //捕获构造异常
      try{
         constructor(resolve,reject);
      }catch(e){
         reject(e);
      }
  }
  ```

  同时，需要在MyPromise的原型上定义链式调用的then方法：

  ```javascript
  MyPromise.prototype.then=function(onFullfilled,onRejected){
     let self=this;
     switch(self.status){
        case "resolved":
          onFullfilled(self.value);
          break;
        case "rejected":
          onRejected(self.reason);
          break;
        default:       
     }
  }
  ```

  **实现步骤**

  （1）核心逻辑实现：resolve和reject函数使得状态改变、接收成功后的值和失败后的原因

  （2）加入异步逻辑：如果是等待状态，就把成功回调函数和失败回调函数存储起来，状态改变时再调用

  （3）实现then方法多次调用添加多个处理函数：异步时最后一个处理函数会覆盖前面的，应该用数组存储

  （4）实现then方法的链式调用：返回promise对象，上一个回调函数的返回值是下一个then方法回调函数的参数；特别处理返回值是promise对象的情况。

  （5）如果onFullfilled函数返回的是该promise本身，那么会抛出类型错误

  https://lagoufed.feishu.cn/drive/folder/fldcnbBcBQwyZYQSJebiDg2jGEf

  TLfl

  

##### 3、介绍一下 promise及其底层如何实现

promise是为了解决异步编程中回调地域问题的一个对象，它的核心实现逻辑是1）每个promise对象有pending、fullfilled、rejected三种状态，resolve方法可以实现pending到fullfilled的状态改变，reject方法可以实现pending到rejected的状态改变，一旦promise对象的状态发生改变，就是不可逆的。2）then方法中的onfullfilled和onrejected函数是promise对象状态发生改变时调用的，成功状态下调用onfullfilled函数，失败状态下调用onrejected函数。3）then方法返回的是一个新的promise对象，以实现then方法的链式调用，避免回调函数嵌套的问题。4）onfullfilled回调函数的返回值可以是普通值也可以是一个promise对象（可以返回标准的Promise对象，还可以返回带有then方法的普通对象/类，即thenable 对象），如果是普通值，直接resolve(普通值)传递；如果是一个promise对象，它的状态将决定then方法返回的promise对象的状态。同时还应该注意promise对象自返回的问题。

加上问题2的回答

##### 4、promise+Generator+Async 的使用

分开解释

- promise：是为了解决异步编程中回调地域问题而存在的一个对象

  promise/A+规范...

  使用: 实例化 promise 对象需要传入函数(包含两个参数),resolve 和 reject,内部确定状态；resolve 和 reject 函数传入的参数在 then 的回调函数中接收。catch 相当于.then(null, rejection) ，当 then 中没有传入 rejection 时,错误会冒泡进入 catch 函数中,若传入了 rejection,则错误会被 rejection 捕获,而且不会进入 catch.此外,then 中的回调函数中发生的错误只会在下一级的 then 中被捕获,不会影响该 promise 的状态. 

- Generator：生成器函数，通常搭配yield使用

  ```javascript
  function* g() {
    var o = 1;
    yield o++;
    yield o++; 
  }
  var gen = g(); // 调用生成器函数，得到生成器对象
  // 调用生成器对象的next方法，函数体才开始执行，遇到yield暂停
  console.log(gen.next()); // Object {value: 1, done: false} 
  var xxx = g(); 
  console.log(gen.next()); // Object {value: 2, done: false}
  console.log(xxx.next()); // Object {value: 1, done: false} 
  console.log(gen.next()); // Object {value: undefined, done: true}
  ```

  generator 和异步控制: 

  利用 Generator 函数的暂停执行的效果，可以把异步操作写在 yield 语句里面，等到调用next 方法时再往后执行。这实际上等同于不需要写回调函数了，因为异步操作的后续操作可以放在 yield 语句下面，反正要等到调用 next 方法时再执行。所以，Generator 函数 

  的一个重要实际意义就是**用来处理异步操作，改写回调函数**。

- Async是为promise存在的语法糖，相当于new Promise()

  async 表示这是一个 async 函数，await 只能用在这个函数里面。 

  await 表示在这里等待异步操作返回结果，再继续执行。 await 后一般是一个 promise 对象

  async 用于定义一个异步函数，该函数返回一个 Promise。 如果 async 函数返回的是一个同步的值，这个值将被包装成一个理解 resolve 的 Promise， 等同于 return Promise.resolve(value)。

  

##### 5、setTimeout 和 Promise 的执行顺序

优先级：主线程>promise产生的微任务>setTimeout产生的宏任务

```javascript
setTimeout(function() {
  console.log(1) }, 0); 
new Promise(function(resolve, reject) {
  console.log(2) 
  for (var i = 0; i < 10000; i++) {
    if(i === 10) {
      console.log(10)
    } 
    i == 9999 && resolve(); 
  }
  console.log(3) 
}).then(function() { 
  console.log(4) 
})
console.log(5);
// 2 10 3 5 4 1
```

```javascript
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}
async function async2() {
  console.log('async2');
}

console.log('illegalscript start');

setTimeout(function() {
    console.log('setTimeout');
}, 0);  
async1();

new Promise(function(resolve) {
    console.log('promise1');
    resolve();
  }).then(function() {
    console.log('promise2');
});

console.log('illegalscript end');

illegalscript start
async1 start
async2
promise1
illegalscript end
async1 end
promise2
setTimeout
```

https://blog.csdn.net/m0_37922914/article/details/108564884?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-1.control&spm=1001.2101.3001.4242

##### 6、如果已经有三个 promise，A、B 和 C，想串行执行，该怎么写？

方法一：

```javascript
// promise队列
A.then(value => {
  console.log(value);
  return B
}).then(value => {
  console.log(value);
  return C
}).then(value => {
  console.log(value);
}).catch(...);
```

方法二：

```javascript
(async () => {
  await A; // 等待Promise状态改变后才会往下执行，否则暂停
  await B;
  await C;
})();
```

##### 7、async 和 await 具体该怎么用？

async相当于new Promise()（ 内部不定义，调用函数直接会返回一个状态成功的Promise对象）

await相当于.then()，后面跟一个Promise对象，表示等待该对象完成，根据该对象的结果继续往下执行

```javascript
(async () = > { 
  await new promise(); 
})()
```

##### 8、promise 和 await/async 的关系

- 关系

  promise和async/await都是处理异步请求

  异步编程的最高境界就是不关心它是否是异步。async、await很好的解决了这一点，将异步强行转换为同步处理。
  async/await与promise不存在谁代替谁的说法，因为async/await是寄生于Promise，Generater的语法糖。

- 用法

  async用于申明一个function是异步的，而await是等待一个异步方法执行完成，而且await只能在async里面使用

  当调用一个 async 函数时，会返回一个 Promise 对象

  await表示在这里等待一个promise返回，再接下来执行，故await后面跟着的应该是一个promise对象，（也可以不是，如果不是接下来也没什么意义了…）

- 区别

  promise是ES6，async/await是ES7
  async/await相对于promise来讲，写法更加优雅
  reject状态：
      1）promise错误可以通过catch来捕捉，建议尾部捕获错误，
      2）async/await既可以用.then又可以用try-catch捕捉

  ```javascript
  let p = new Promise((resolve,reject) => {
      setTimeout(() => {
          reject('error');
      },1000);
  });
  
  async function demo(params) {
      try {
          let result = await p;
      }catch(e) {
          console.log(e);
      }
  }
  
  demo();
  ```

  ##### 9、说 promise，没有 promise 怎么办

  回调函数

  generator实现异步

### 【跨域】

#####   1、跨域原理

跨域，是指浏览器不能执行其他网站的脚本。它是由浏览器的同源策略造成的;

同源策略是浏览器对 JavaScript 实施的安全限制，只要**协议、域名、端口**有任何一个不同，都被当作是不同的域。跨域原理，即是通过各种方式，避开浏览器的安全限制。 

- 同源策略限制以下几种行为：

  ```javascript
  1.) Cookie、LocalStorage 和 IndexDB 无法读取
  2.) DOM 和 Js对象无法获得
  3.) AJAX 请求不能发送
  ```

##### 2、JS 实现跨域 --- 9种

https://segmentfault.com/a/1190000011145364

**跨域解决方案**

- **通过jsonp跨域**

  JSONP用的是script标签，通过src属性向服务器发请求，同时把本地的一个函数传递给服务器，返回时即执行全局函数；与Ajax提供的XMLHttpRequest没有关系

  **缺点**：只能实现get请求（参数只能在src的？后传递）;

  ​			JSONP 需要服务器端配合返回指定格式的数据。 

  1.）原生实现：

  ```javascript
   <script>
      var script = document.createElement('script');
      script.type = 'text/javascript';
  
      // 传参一个回调函数名给后端，方便后端返回时执行这个在前端定义的回调函数
      script.src = 'http://www.domain2.com:8080/login?user=admin&callback=handleCallback';
      document.head.appendChild(script);
  
      // 回调执行函数
      function handleCallback(res) {
          alert(JSON.stringify(res));
      }
   </script>
  ```

  服务端返回如下（返回时即执行全局函数）：

  ```javascript
  handleCallback({"status": true, "user": "admin"})
  ```

  2.）jquery ajax：

  ```javascript
  $.ajax({
      url: 'http://www.domain2.com:8080/login',
      type: 'get',
      dataType: 'jsonp',  // 请求方式为jsonp
      jsonpCallback: "handleCallback",    // 自定义回调函数名
      data: {}
  });
  ```

- **document.domain + iframe跨域**

  此方案仅限主域相同，子域不同的跨域应用场景。

  实现原理：两个页面都通过js强制设置document.domain为基础主域，就实现了同域。

  默认情况下，document.domain存放的是载入文档的服务器的主机名，可以手动设置这个属性，不过是有限制的，只能设置成当前域名或者上级的域名，并且必须要包含一个.号，也就是说不能直接设置成顶级域名。例如：id.qq.com，可以设置成qq.com，但是不能设置成com。具有相同document.domain的页面，就相当于是处在同域名的服务器上，如果协议和端口号也是一致，那它们之间就可以跨域访问数据。

  1.）父窗口：([http://www.domain.com/a.html)](https://link.segmentfault.com/?url=http%3A%2F%2Fwww.domain.com%2Fa.html))

  ```javascript
  <iframe id="iframe" src="http://child.domain.com/b.html"></iframe>
  <script>
      document.domain = 'domain.com';
      var user = 'admin';
  </script>
  ```

  2.）子窗口：([http://child.domain.com/b.html)](https://link.segmentfault.com/?url=http%3A%2F%2Fchild.domain.com%2Fb.html))

  ```javascript
  <script>
      document.domain = 'domain.com';
      // 获取父窗口中变量
      alert('get js data from parent ---> ' + window.parent.user);
  </script>
  ```

-  **location.hash + iframe**

  实现原理： a欲与b跨域相互通信，通过中间页c来实现。 三个页面，不同域之间利用iframe的location.hash传值，相同域之间直接js访问来通信。

- **window.name + iframe跨域**

  window.name属性的独特之处：name值在不同的页面（甚至不同域名）加载后依旧存在，并且可以支持非常长的 name 值(2MB)

  通过iframe的src属性由外域转向本地域，跨域数据即由iframe的window.name从外域传递到本地域。这个就巧妙地绕过了浏览器的跨域访问限制，但同时它又是安全操作。

- **postMessage跨域**

  可以跨域操作的 window 属性之一。

  用法：postMessage(data,origin)方法接受两个参数
  data： html5规范支持任意基本类型或可复制的对象，但部分浏览器只支持字符串，所以传参时最好用JSON.stringify()序列化。
  origin： 协议+主机+端口号，也可以设置为"*"，表示可以传递给任意窗口，如果要指定和当前窗口同源的话设置为"/"。

  1.）a.html：(http://www.domain1.com/a.html)

  ```javascript
  // 利用iframe向b发送请求
  <iframe id="iframe" src="http://www.domain2.com/b.html" style="display:none;"></iframe>
  <script>       
      var iframe = document.getElementById('iframe');
      iframe.onload = function() {
          var data = {
              name: 'aym'
          };
          // iframe.contentWindow是b页面的全局对象window
          // 向domain2传送跨域数据
          iframe.contentWindow.postMessage(JSON.stringify(data), 'http://www.domain2.com');
      };
  
      // 接受domain2返回数据
      window.addEventListener('message', function(e) {
          alert('data from domain2 ---> ' + e.data);
      }, false);
  </script>
  ```

  2.）b.html：(http://www.domain2.com/b.html)

  ```javascript
  <script>
      // 接收domain1的数据
      window.addEventListener('message', function(e) {
          alert('data from domain1 ---> ' + e.data);
  
          var data = JSON.parse(e.data);
          if (data) {
              data.number = 16;
  
              // 处理后再发回domain1
              // e.source指A
              e.source.postMessage(JSON.stringify(data), 'http://www.domain1.com');
          }
      }, false);
  </script>
  ```

  

- **跨域资源共享（CORS）**

  普通跨域请求：只服务端设置Access-Control-Allow-Origin即可，前端无须设置，若要带cookie请求：前后端都需要设置。

  **缺点：允许的源只有一个，或者*开放，但是开放时不允许请求携带cookie**

  **1、 前端设置：**

  1.）原生ajax

  ```javascript
  // 前端设置是否带cookie
  xhr.withCredentials = true;
  ```

  实例代码

  ```javascript
  var xhr = new XMLHttpRequest(); // IE8/9需用window.XDomainRequest兼容
  
  // 前端设置是否带cookie
  xhr.withCredentials = true;
  
  xhr.open('post', 'http://www.domain2.com:8080/login', true);
  xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
  xhr.send('user=admin');
  
  xhr.onreadystatechange = function() {
      if (xhr.readyState == 4 && xhr.status == 200) {
          alert(xhr.responseText);
      }
  };
  ```

- **nginx代理跨域**

  不需要前端干啥

- **nodejs中间件代理跨域**

  node中间件实现跨域代理，原理大致与nginx相同，都是通过启一个代理服务器，实现数据的转发

  **proxy相当于node给你模拟了一个nginx服务请求，服务器请求服务器不存在跨域，proxy请求回来数据，解决跨域**

  #####  **vue框架的跨域（1次跨域）**

  利用node + webpack + webpack-dev-server代理接口跨域。

  webpack.config.js部分配置：

  ```javascript
  module.exports = {
      entry: {},
      module: {},
      ...
      devServer: {
          historyApiFallback: true,
          proxy: [{
              context: '/login',
              target: 'http://www.domain2.com:8080',  // 代理跨域目标接口
              changeOrigin: true,
              secure: false,  // 当代理某些https服务报错时用
              cookieDomainRewrite: 'www.domain1.com'  // 可以为false，表示不修改
          }],
          noInfo: true
      }
  }
  ```

- **WebSocket协议跨域**

  通过socket.io实现

**补充知识点**

- 协议及对应端口

  网络技术中的端口默认指的是TCP/IP协议中的服务端口，一共有0-65535个端口。

  http 80、https 443、ftp 21、

- IP、域名、端口号之间联系

  域名通常会和IP进行绑定，通过访问域名来访问网络上的主机的服务。IP地址通常指的是网络中的主机，而域名则通常表示一个网站，一个域名可以绑定到多个ip上，多个域名也可以绑定到一个ip上。

  端口（英语：port），主要分为物理端口和逻辑端口。我们一般说的都是逻辑端口，用于区分不同的服务。因为网络中一台主机只有一个IP，但是一个主机可以提供多个服务，端口号就用于区分一个主机上的不同服务。一个IP地址的端口通过16bit进行编号，最多可以有65536个端口，标识是从0~65535。

  端口号分为公认端口（0~1023）、注册端口（1024~49151）和动态端口（49152~65535）。我们自己的服务一般都绑定在注册端口上

  https://blog.csdn.net/fightsyj/article/details/86482820

##### 3、跨域（jsonp，ajax）

JSONP：ajax 请求受同源策略影响，不允许进行跨域请求，而 script 标签 src 属性中的链接却可以访问跨域的 js 脚本，利用这个特性，服务端不再返回 JSON 格式的数据，而是返回一段调用某个函数的 js 代码，调用函数处理数据，这样实现了跨域。

##### 4、axios 是什么？怎样使用它？怎么解决跨域的问题？

- axios是基于promise的ajax封装库
- 安装 npm install axios --save 即可使用，请求中包括 get,post,put, patch ,delete 等五种请求方式
- 解决跨域可以在请求头中添加Access-Control-Allow-Origin，也可以在 index.js 文件中更改 proxyTable 配置等解决跨域问题。





### 【浏览器】

##### 1、cookie

1）cookie是是由服务器生成、保存在客户端的一种信息载体。这个载体中存放着用户访问该站点的会话状态信息。只要cookie没被清空/没有失效，那么保存在其中的会话状态就有效。

2）cookie由若干键值对组成。客户端第一次发送请求时，服务器端生成cookie，保存在响应头中发送给客户端，客户端存在在本地，当再次发送同类的请求（同域）时，请求中会自动添加所保留的cookie信息，服务器端会对cookie解析进行操作。

3）cookie的存储位置取决于服务器端的设置：客户端的硬盘/浏览器的缓存。

4）cookie禁用：指客户端不接收，但服务端可以生成并保存在响应头中发送给客户端

##### 2、session

1）session是由服务器生成、保存在服务器端的一种会话跟踪技术。（会话：一组请求和响应）

2）session工作原理

服务器端会为每个会话维护一个session对象。

- 写入session列表

  服务器端的session是以Map的形式进行管理，用户第一次提交请求时，服务器创建session对象并写入列表

  key：32位随机字符串    value：session对象的引用

- 服务器生成并发送cookie

  将sessionID作为name，32位随机字符串作为value，以cookie的形式存放在响应报头中，发送给客户端

- 客户端接收并发送cookie

  客户端接收后将cookie存储到浏览器缓存中，只要浏览器不关闭，cookie就不消失

  当客户端再次发出请求时将cookie数据添加到请求头部，发送到服务器端

- 从session列表中查找

  服务器端根据cookie中的sessionID在map中查找对应的session对象

3）cookie禁用后的session会话跟踪

服务器端会以为客户端的每次请求都是新的（不带sessionID），每次请求都创建session对象；

会话的结束：

​		对于浏览器：关闭

​		对于服务器：session的失效

- cookie禁用后重定向时的session会话跟踪
- cookie禁用后非重定向时的session会话跟踪

4）cookie/session/localStorage/seesionStorage

|          | Cookie               | localStorage               | sessionStorage                 | Session              |
| -------- | -------------------- | -------------------------- | ------------------------------ | -------------------- |
|          | k-v存储              | k-v存储                    | k-v存储                        | k-v存储              |
| 存储位置 | 客户端               | 客户端                     | 客户端                         | 服务端               |
| 特点     | 随请求头每次提交     | 不随头提交<br />可长时保存 | 不随头提交<br />页面关闭即失效 | 安全                 |
| 跨页     | 可跨页<br />不可跨域 | 可跨页<br />不可跨域       | 不可跨页<br />不可跨域         | 可跨页<br />不可跨域 |

##### 3、Cookie、sessionStorage、localStorage 的区别

共同点：都是保存在浏览器端，并且是同源的

|           | Cookie                                                       | localStorage                                   | sessionStorage                                       |
| --------- | ------------------------------------------------------------ | ---------------------------------------------- | ---------------------------------------------------- |
| 生命周期  | 可设置失效时间，否则默认为关闭浏览器后失效                   | 除非被手动清除，否则永久保存                   | 仅在当前网页会话下有效，关闭页面或浏览器后就会被清除 |
| 存放数据  | 4k 左右                                                      | 可以保存 5M 的信息                             | 可以保存 5M 的信息                                   |
| http 请求 | 每次都会携带在 http 头中，在浏览器和服务器间来回传递；如果使用 cookie 保存过多数据会带来性能问题 | 仅在客户端即浏览器中保存，不参与和服务器的通信 | 仅在客户端即浏览器中保存，不参与和服务器的通信       |
| 易用性    | 需要程序员自己封装，原生的 cookie 接口不友好                 | 可采用原生接口，亦可再次封装                   | 可采用原生接口，亦可再次封装                         |

从安全性来说，因为每次 http 请求都回携带 cookie 信息，这样子浪费了带宽，所以 cookie应该尽可能的少用，此外 cookie 还需要指定作用域，不可以跨域调用，限制很多，但是用户识别用户登陆来说，cookie还是比storage好用，其他情况下可以用storage，localstorage 可以用来在页面传递参数，sessionstorage 可以用来保存一些临时的数据，防止用户刷新页面后丢失了一些参数。

##### 4、cookie和session的区别

HTTP 是一个无状态协议，因此 Cookie 的最大的作用就是存储 sessionId 用来唯一标识用户。

- cookie 数据存放在客户的浏览器上，session 数据放在服务器上。 
- cookie 不是很安全，别人可以分析存放在本地的 COOKIE 并进行 COOKIE 欺骗，考虑到安全应当使用 session。 
-  session 会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能，考虑到减轻服务器性能方面，应当使用 COOKIE。 
-  单个 cookie 保存的数据不能超过 4K，很多浏览器都限制一个站点最多保存 20 个cookie。

##### 5、cookie 有哪些字段可以设置

##### 6、cookie 有哪些编码方式？









