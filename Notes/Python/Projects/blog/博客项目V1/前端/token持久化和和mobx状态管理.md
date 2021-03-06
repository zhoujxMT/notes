[TOC]

## token持久化

使用LocalStorage来存储token。

LocalStorage是浏览器端持久化方案之一，HTML5标准增加的技术。

LocalStorage是为了存储得到的数据，例如Json。

数据存储就是键值对。

数据会存储在不同的域名下面，不同浏览器对单个域名下存储数据的长度支持不同，有的最多支持2MB。

![img](token%E6%8C%81%E4%B9%85%E5%8C%96%E5%92%8C%E5%92%8Cmobx%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86.assets/bfdd03cf-46aa-4aed-b3df-8063082937bf.png)

SessionStorage：和LocalStorage差不多，它是会话级的，浏览器关闭，会话结束，数据清除。

IndexedDB：一个域一个datatable；key-value检索方式；建立在关系型的数据模型之上，具有索引、游标、事务等概念。

### store.js

store.js是一个兼容所有浏览器的LocalStore包装器，不需要借助Cookie或Flash。它会根据浏览器自动选择使用LocalStorage、GlobalStorage或userData来实现本地存储功能。

安装

```shell
npm i store
```



测试代码

```
let store = require('store');
store.set('user', 'XiaoFei');
console.log(1,store.get('user'));

store.remove('user');
console.log(2, store.get('user'));
console.log(3, store.get('user', 'a'));

store.set('user', {'name':'XiaoFei', 'age':20});
console.log(4, store.get('user').name);
console.log(5, store.get('user').age);

store.set('school', {'name':'magedu'});
store.each(function(value, key){  // key,value是反的
    console.log(6, key, '<--->', value);
});

store.clearAll();
console.log(7, store.get('user'));
```

安装store的同时，也安装了expire过期插件，可以在把kv对存储到LS中的时候顺便加入过期时长。

```
store.addPlugin(require('store/plugins/expire'));  // 过期插件
store.set('token', res.data.token, (new Date()).getTime() + (8*3600*1000));
```



## Mobx状态管理

Redux和Mobx

社区提供的状态管理库，有Redux和Mobx。

Redux代码优秀，使用严格的函数式编程思想，但学习曲线陡峭，比较难学，小项目使用的优势不明显。

Mobx优秀稳定，简单方便，适合中小项目使用，使用面向对象方式，简单易学，目前在项目中使用非常广泛。

Mobx和React是一对强力组合。

Mobx官方网站：https://mobx.js.org/README.html

Mobx中文官网：https://cn.mobx.js.org/

Mobx由Mendix、Coinbase、Facebook开源，它实现了观察者模式。

观察者模式：也称为发布订阅模式，观察者观察某个目标，目标对象(Obseralbe)状态发生了变化，就会通知自己内部注册了的观察者Observer。

### 状态管理

需求：一个组件的onClick触发事件响应函数，此函数会调用后台服务。但是后台服务比较耗时，等处理完，需要引起组件的渲染操作。也就是说通过异步方式，提交了一个事件，不会等着它处理结束，而是根据它处理的结果来做相应变化。

要渲染组件，需要使用改变组件的props或state。

observable装饰器：设置被观察者

observer装饰器：设置观察者，将React组件转换为响应式组件。

### 示例代码

src/component/user.js

```
import axios from "axios";
import store from 'store';
import {observable} from 'mobx';

// 加载过期插件
store.addPlugin(require('store/plugins/expire'));

export default class UserService {
    @observable loggedin = false;  // 被观察对象

    login(email, password) {
        console.log('>>>>> ', email, password)
        // axios异步
        axios.post(
            '/api/user/login',
            {
                'email': email,
                'password':password
            }
        ).then(
            response => {
                console.log(response, '11111');
                console.log(response.data, '22222');
                console.log(response.status, '33333');

                // obj.setState({ret:1000});

                store.set(  // 存储token
                    'token',response.data.token,
                    (new Date()).getTime() + (8*3600*1000)
                );
                this.loggedin = true;  // ret发生变化
                
            }
        ).catch(
            error => {
                console.log(error);
            }
        );
    }
}
```

src/component/login.js

```js
import React from 'react';
import {
    Link,
    Redirect
} from "react-router-dom";
import "../css/login.css";
import UserService from "../service/user"
import { observer } from 'mobx-react';

const service = new UserService();

export default class Login extends React.Component {
    render() {
        return <_Login service={service} />;
    }
}

@observer  // 观察者
class _Login extends React.Component {
    // state = {'ret': -1}
    handleClick(event) {
        event.preventDefault();
        console.log(1, event.target)
        console.log(2, event.target.form)
        let fm = event.target.form;
        this.props.service.login(
            fm[0].value,
            fm[1].value,
        );
        console.log('in login.js ~~~~~', fm[0].value, fm[1].value);
    }
    render() {
        // if (this.state.ret != -1) {
        //     console.log(this.state.ret, '---------');
        //     return (<Redirect to='/about' />);
        // }
        if (this.props.service.loggedin) {
            console.log('observer loging ~~~~~~~~');
            return <Redirect to='/' />;
        }
        
        return (
            <div className="login-page">
                <div className='form'>
                    <form className="login-form">
                        <input type="text" placeholder="邮箱" />
                        <input type="password" placeholder="密码" />
                        <button onClick={this.handleClick.bind(this)}>登录</button>
                        <p className="message">
                            没有注册?
                            <Link to="reg">
                                请注册
                            </Link>
                        </p>
                    </form>
                </div>
            </div>
        );
    }
}
```

需要注意：

Service中被观察者loggedin变化，导致了观察者调用render函数。在观察者的render函数中，一定要使用这个被观察对象。

测试时，开启Django后台服务。