
# 第六章 Vuex 状态管理

　　书说至此，我们编写的组件的所有数据都是存放在组件内部的。我们调用一个 API，然后把返回的数据存放在一个数据对象中。我们把一个表单绑定到一个对象上，还是把这个对象存放在这个数据对象中。组件之间的所有通信都采用事件（events）方式（子组件往父组件通信）和属性（props）方式（父组件往子组件通信）。在简单的应用场合中，这一套用着不错，但在稍微复杂一点的应用中，这就捉襟见肘了。

　　让我们用一个社交网络应用来举个例，就说其中的消息部分。比如说，你想在应用顶部导航栏上放一个图标，用来显示你收到的消息数量，同时在页面底部，你还想要一个消息弹窗，同样是告诉你收到的消息数量。因为图标和弹窗这两个组件彼此在页面上并无直接联系，所以用 events 和 props 来连接它们就会一团糟：与消息通知无关的组件将不得不传递这些额外的事件（译注：因为那两个与消息通知有关的组件之间没有直接联系，并非父子关系，如果用事件方式或属性方式通信，则必然要经过其它组件传递事件或属性）。另外一种方法是，不通过连接两个组件的方式来共享数据，而是给每个组件都单独做一个 API。这么做就更糟了：不同的组件将会在不同的时间点更新，这就意味着它们会渲染不一样的数据，并且页面所发送的 API 请求也会远远超过其实际所需。  

　　*vuex* 应运而生，帮助开发者管理 Vue 应用中的状态。它提供了一种集中式存储（centralized store），你可以在整个应用中使用它来存储和维护全局状态。它同时还使你能够对存入的数据进行校验，以保证当这个数据再次被取出时是可预见而且正确的。

## 如何安装
　　你可以用 CDN 来引入 vuex，只需加入如下代码：
```javascript
<script src="https://unpkg.com/vuex"></script>
```

　　此外，如果你正在使用 npm，你也可以通过 `npm install --save vuex` 来安装 vuex。如果你正在使用的是诸如 webpack 的打包工具，那么就要像使用 vue-router 那样，调用 `Vue.use()`：
```javascript
import Vue from 'vue'; 
import Vuex from 'vuex';
Vue.use(Vuex);
```

　　然后就是创建你的 store。我们创建并保存 *store/index.js* 文件，内容如下：

```javascript
import Vuex from 'vuex';

export default new Vuex.Store({
  state: {}
});
```


　　现在这个 store 还空无一物，我们这一整章都会往里面加东西。  

　　接着，将它引入到你的主应用文件中，同时在 Vue 实例化时作为一个属性传入。

```javascript
import Vue from 'vue';
import store from './store';

new Vue({
  el: '#app',
  store,
  components: {
    App
  }
});
```
　　现在，你已经把这个 store 加到你的应用中了，并且可以用 `this.$store` 来访问它。  

　　接下来我们先来看看有关 vuex 的一些概念，然后我们再来看看能用 `this.$store` 做些什么。

## 概念
　　正如本章开头所提到的，vuex 可以满足复杂应用中多个组件进行状态共享的需求。  

　　让我们用一个简单的组件来说明一下，这个组件显示的是用户在页面上所拥有的消息数目，它不使用 vuex。

```javascript
const NotificationCount = {
  template: `<p>Messages: {{ messageCount }}</p>`,
  data: () => ({
    messageCount: 'loading'
  }),
  mounted() {
    const ws = new WebSocket('/api/messages');

    ws.addEventListener('message', (e) => {
      const data = JSON.parse(e.data);
      this.messageCount = data.messages.length;
    });
  }
};
```

　　这个组件相当简单。它打开了一个 websocket 通道到 */api/message* 这个地址，然后当服务器发送数据给客户端时——在本例中，就是当 socket 打开时（发送消息的初始数目），以及当这个数目更新时（也就是有新消息时）——在这个 socket 上发送的消息数量就会被显示在页面上。

>　　在实际应用中，这份代码会更复杂。因为在这个例子中并没有进行 websocket 认证，并且假设了在 websocket 上获得的服务器响应总是一个合法的 JSON，同时这个 JSON 带有一个 `messages` 属性，并且这个 `messages` 属性是一个数组。实际情况可能与此相悖，但就例子而言，我们就使用简化的代码来完成工作。

　　当我们想要在同一个页面上使用不止一个 `NotificationCount` 组件时，问题就来了。因为每个组件都会打开一个 websocket，所以就创建了一些不必要的重复连接，并且由于网络延迟，各个组件的更新时间可能会稍稍有点不同。为了解决这个问题，我们可以把 websocket 的逻辑放入 vuex 中。

　　让我们立即通过一个例子来深入了解。我们修改一下组件：

```javascript
const NotificationCount = {
  template: `<p>Messages: {{ messageCount }}</p>`,
  computed: {
    messageCount() {
      return this.$store.state.messages.length;
    }
  }
  mounted() {
    this.$store.dispatch('getMessages');
  }
};
```

　　然后下面是我们的 vuex store （store/index.js）：

```javascript
let ws;

export default new Vuex.Store({
  state: {
    messages: [],
  },
  mutations: {
    setMessages(state, messages) {
      state.messages = messages;
    }
  },
  actions: {
    getMessages({ commit }) {
      if (ws) {
        return;
      }

      ws = new WebSocket('/api/messages');

      ws.addEventListener('message', (e) => {
        const data = JSON.parse(e.data);
        commit('setMessages', data.messages);
      });
    }
  }
});
```

　　现在，虽然每个已挂载的通知计数组件都会触发 `getMessages` 动作，但是这个动作会检查是否存在 websocket 实例，并且只在无连接的时候创建连接。连接后，它就开始监听这个连接套接字（socket），提交状态变更，这个变更会在所有的通知计数组件上得以反映，这是因为 store 是响应式的——正如 Vue 中大多数的其它事物那样。当这个套接字收到新的数据时，这个全局 store 就会跟着更新，同时这个页面上的每个组件也都会跟着更新。  

　　在本章的剩余部分，我会引入一些单独的概念，并把它们加入到例子当中，它们就是 状态（state）, 变更（mutations），和动作（action），同时我还会阐述大型应用中构建 vuex 模块的方法，以免这些模块形成一个过分庞大而且混乱的文件。  

## State 及其辅助函数

　　首先，我们来看看 state。state 表示数据在 vuex 中的存储状态，它就像一个我们在应用的任何角落都能访问到的庞大对象——是的，它就是*单一数据源（single source of truth）*。

　　让我们来看看仅存储了一个数字的 store:

```javascript
import Vuex from 'vuex';

export default new Vuex.Store({
  state: {
    messageCount: 10
  }
});
```

