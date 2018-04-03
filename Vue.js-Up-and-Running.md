
# 0

　　书说至此，我们所有组件的数据都是存放在组件内部的。我们使用一个API，并将返回的数据存放在一个数据对象中。我们把一个表单绑定到一个对象，并把这个对象存放在这个数据对象中。组件之间所有的通信都采用事件方式（子组件往父组件）和传参方式（父组件往子组件）。在简单的应用场合中，这一套用着不错，但在稍微复杂一点的应用中，这就捉襟见肘了。  
　　让我们用一个社交网络应用来举个例，就说其中的消息部分。比如说，你想在应用顶部导航栏上放一个图标，用来显示你收到的消息数量，同时在页面底部，你还想要一个消息弹窗，同样是告诉你收到的消息数量。因为图标和弹窗这两个组件彼此在页面上并不相近，所以用 events 和 props 来连接它们就会是一团糟：与通知无关的组件将不得不关注传入给它们的带有通知的事件（译注：因为那两个与通知有关的组件如果用事件方式或传参方式通信，则必然要经过其它组件传递事件或参数）。还有一种方法是，不通过把这两个组件连接在一起的方法来共享数据，而是给每个组件都单独做一个API。这么做更糟！不同的组件将会在不同的时间更新，这就意味着它们会显示不一样的东西，并且页面所使用的API会远远超过其实际所需。  
　　vuex 应运而生，帮助开发者管理Vue应用中的状态。它提供了一块集中的存储区域，你可以在整个应用中使用它来存储和访问全局状态。它同时还使你能够对存入的数据进行校验，以保证当这个数据再次被取出时是可预见而且正确的。

## 1
　　你可以用过 CDN 来引入 vuex，添加如下标签：
```javascript
<script src="https://unpkg.com/vuex"></script>
```
　　此外，如果你正在使用 npm，你也可以通过 `npm install --save vuex` 来安装 vuex。如果你正在使用的是诸如 webpack 的打包工具，那么就要像使用 vue-router 那样，调用 `Vue.use()`：
```javascript
import Vue from 'vue'; 
import Vuex from 'vuex';
Vue.use(Vuex);
```
　　然后就是建立你的存储区。创建文件 store/index.js 并保存如下内容：
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
　　接下来我们先来看看有关 vuex 的一些概念，然后我们再看看能用 `this.$store` 做些什么。

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
