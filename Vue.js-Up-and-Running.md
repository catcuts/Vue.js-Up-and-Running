
# 0

Until this point in the book, all data has been stored in our components. We hit an
API, and we store the returned data on the data object. We bind a form to an object,
and we store that object on the data object. All communication between components
has been done using events (to go from child to parent) and props (to go from parent
to child). This is good for simple cases, but in more complicated applications, it won’t
suffice.  
　　书说至此，我们所有组件的数据都是存放在组件内部的。我们调用一个 API ，然后把返回的数据存放在一个数据对象中。我们把一个表单绑定到一个对象，还是把这个对象存放在这个数据对象中。组件之间所有的通信都采用事件（events）方式（子组件往父组件）和属性（props）方式（父组件往子组件）。在简单的应用场合中，这一套用着不错，但在稍微复杂一点的应用中，这就捉襟见肘了。

Let’s take a social network app—specifically, messages. You want an icon in the top
navigation to display the number of messages you have, and then you want a mes‐
sages pop-up at the bottom of the page that will also tell you the number of messages
you have. Both components are nowhere near each other on the page, so linking
them using events and props would be a nightmare: components that are completely
unrelated to notifications will have to be aware of the events to pass them through.
The alternative is, instead of linking them together to share data, you could make sep‐
arate API requests from each component. That would be even worse! Each compo‐
nent would update at different times, meaning they could be displaying different
things, and the page would be making more API requests than it needed to.  
　　让我们用一个社交网络应用来举个例，就说其中的消息部分。比如说，你想在应用顶部导航栏上放一个图标，用来显示你收到的消息数量，同时在页面底部，你还想要一个消息弹窗，同样是告诉你收到的消息数量。因为图标和弹窗这两个组件彼此在页面上并无直接联系，所以用 events 和 props 来连接它们就会是一团糟：与消息通知无关的组件将不得不传递这些额外的事件（components that are completely unrelated to notifications will have to be aware of the events to pass them through.）（译注：因为那两个与消息通知有关的组件没有直接联系，并非父子关系，如果用事件方式或属性方式通信，则必然要经过其它组件传递事件或属性）。另外一种方法是，不通过连接两个组件的方式来共享数据，而是给每个组件都单独做一个API。这么做就更糟了。不同的组件将会在不同的时间点更新，这就意味着它们会渲染不一样的数据，并且页面所发送的 API 请求也会远远超过其实际所需。  

vuex is a library that helps developers manage their application’s state in Vue applica‐
tions. It provides one centralized store that you can use throughout your app to store
and work with global state, and gives you the ability to validate data going in to
ensure that the data coming out again is predictable and correct.  
　　*vuex* 应运而生，帮助开发者管理 Vue 应用中的状态。它提供了一种集中式存储，你可以在整个应用中使用它来存储和维护全局状态。它同时还使你能够对存入的数据进行校验，以保证当这个数据再次被取出时是可预见而且正确的。

## 1
　　你可以用过 CDN 来引入 vuex，只需加入如下代码：
```javascript
<script src="https://unpkg.com/vuex"></script>
```
　　此外，如果你正在使用 npm，你也可以通过 `npm install --save vuex` 来安装 vuex。如果你正在使用的是诸如 webpack 的打包工具，那么就要像使用 vue-router 那样，调用 `Vue.use()`：
```javascript
import Vue from 'vue'; 
import Vuex from 'vuex';
Vue.use(Vuex);
```
Then you need to set up your store. Let’s create the following file and save it as store/index.js:  
　　然后就是创建你的 store。我们创建并保存 store/index.js 文件，内容如下：
```javascript
import Vuex from 'vuex';

export default new Vuex.Store({
  state: {}
});
```
　　现在这个存储区还空无一物，我们这一整章都会往里面加东西。  
　　接下来，将它引入到你的主应用文件中，并将它作为Vue实例化时的一个属性。
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
　　现在，你已经把这个存储加到你的应用中了，并且可以用 `this.$store` 来访问它。  
　　接下来我们先来看看有关 vuex 的一些概念，然后我们再来看看能用 `this.$store` 做些什么。

## 2
As mentioned in the introduction to this chapter, vuex can be required when complex
applications require more than one component to share state.  
　　正如本章一开始所提到的，当复杂应用要求不止一个组件共享状态时，vuex 就显得很有必要。  
Let’s take a simple component written without vuex that displays the number of messages a user has on the page:  
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
It’s pretty simple. It opens a websocket to /api/messages, and then when the server
sends data to the client—in this case, when the socket is opened (initial message
count) and when the count is updated (on new messages)—the messages sent over
the socket are counted and displayed on the page.  
　　这个组件相当简单。它打开了一个 websocket 通道到 `/api/message`，然后当服务器发送数据给客户端时——在本例中，就是当套接字打开时（发送消息的初始数目），以及当这个数目更新时（也就是有新消息时）——在这个套接字上发送的消息数量就会被显示在页面上。

>In practice, this code would be much more complicated: there’s no
authentication on the websocket in this example, and it is always
assumed that the response over the websocket is valid JSON with a
messages property that is an array, when realistically it probably
wouldn’t be. For this example, this simplistic code will do the job.  
在实际应用中，这份代码会更复杂。因为在这个例子中并没有进行 websocket 认证，并且假设了在 websocket 上获得的服务器响应总是一个有效的 JSON，这个 JSON 带有一个 messages 属性，而且这个 messages 属性是一个数组——尽管在实际中可能并非如此。对这个例子而言，我们使用简化的代码来完成工作。

We run into problems when we want to use more than one of the Notification
Count components on the same page. As each component opens a websocket, it
opens unnecessary duplicate connections, and because of network latency, the com‐
ponents might update at slightly different times. To fix this, we can move the web‐
socket logic into vuex.  
当我们想要在同一个页面上使用不止一个 NotificationCount 组件时，问题就来了。因为每个组件都会打开一个 websocket，所以就创建了一些不必要的重复连接，并且由于网络延迟，各个组件的更新时间可能会稍稍有点不同。为了解决这个问题，我们可以把 websocket 的逻辑放入 vuex 中。

Let’s dive right in with an example. Our component will become this:
让我们立即通过一个例子来深入了解。我们的组件将会变成这样：
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
And the following will become our vuex store:
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
Now, every notification count component that is mounted will trigger getMessages,
but the action checks whether the websocket exists and opens a connection only if
there isn’t one already open. Then it listens to the socket, committing changes to the
state, which will then be updated in the notification count component as the store is
reactive—just like most other things in Vue. When the socket sends down something
new, the global store will be updated, and every component on the page will be upda‐
ted at the same time.  
现在，虽然每个挂载的 NotificationCount 组件都会触发 getMessages 动作，但是这个动作会检查是否存在 websocket 实例，并且只在无连接的时候创建连接。连接后，它就开始监听这个连接套接字(socket)，提交状态的变更，这个变更会在 NotificationCount 组件上的已更新，因为 store 是响应式的——正如 Vue 中大多数的其它事物那样。当这个套接字收到新的数据时，这个全局 store 就会跟着更新，同时这个页面上的每个组件也都会跟着更新。  

Throughout the rest of the chapter, I’ll introduce the individual concepts you saw in
that example—state, mutations, and actions—and explain a way we can structure our
vuex modules in large applications to avoid having one large, messy file.  
在本章的末尾，我会引入一些单独的概念，并把它们加入到例子当中，它们就是 状态(state), 变更(mutations)，以及动作(action)，同时我还会阐述大型应用中构建 vuex 模块的方法，以免这些模块形成一个过分庞大而且混乱的文件。  

## State and State Helpers
First, let’s look at state. State indicates how data is stored in our vuex store. It’s like one
big object that we can access from anywhere in our application—it’s the single source
of truth.
首先，我们来看看 state。state 表示数据在 vuex 中的存储状态。它就像一个我们在应用的任何角落都横访问到的庞大对象——是的，它就是单一数据源。

Let’s take a simple store that contains only a number:  
让我们看看仅存储了一个数字的 store:
```javascript
import Vuex from 'vuex';

export default new Vuex.Store({
  state: {
    messageCount: 10
  }
});
```
