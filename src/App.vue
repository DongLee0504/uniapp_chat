<script>
import store from "./store";
import http from "./common/request";
import * as enums from "./common/enums";
import * as wsApi from "./common/wssocket";

export default {
  data() {
    return {
      audioTip: null,
    };
  },
  computed: {
    chatStore() {
      return this.$store.state.chatStore;
    },
    unreadCount() {
      let count = 0;
      this.chatStore.chats.forEach((chat) => {
        count += chat.unreadCount;
      });
      return count;
    },
  },
  watch: {
    unreadCount(newCount, oldCount) {
      const tabBarPages = ["pages/chat/chat", "pages/friend/friend", "pages/group/group", "pages/home/home"];
      const currentPage = getCurrentPages().pop();
      if (tabBarPages.includes(currentPage.route)) {
        this.refreshUnreadBadge(newCount);
      }
    },
  },
  methods: {
    init() {
      // 加载数据
      store
        .dispatch("load")
        .then(() => {
          // 审核
          // this.initAudit();
          // 初始化websocket
          this.initWebSocket();
          // 加载离线消息
          // this.loadPrivateMessage(store.state.chatStore.privateMsgMaxId);
          // this.loadGroupMessage(store.state.chatStore.groupMsgMaxId);
        })
        .catch((e) => {
          console.log(e);
          // this.exit();
        });
    },
    initWebSocket() {
      const sLoginInfo = uni.getStorageSync("zhixin_applets_loginInfo");
      let oLoginInfo = {};
      if (sLoginInfo) {
        oLoginInfo = JSON.parse(sLoginInfo);
      }
      wsApi.connect(import.meta.env.VITE_WS_URL, oLoginInfo.sUserId);
      wsApi.init(oLoginInfo.sUserId);
      wsApi.onMessage((cmd, msgInfo) => {
        if (cmd == 2) {
          // 异地登录，强制下线
          uni.showModal({
            content: "您已在其他地方登陆，将被强制下线",
            showCancel: false,
          });
          this.exit();
        } else if (cmd == 3) {
          if (msgInfo.type == 4) {
            store.dispatch("loadNewFriend");
          } else {
            // 私聊消息
            this.handlePrivateMessage(msgInfo);
          }
        } else if (cmd == 4) {
          // 群聊消息
          this.handleGroupMessage(msgInfo);
        } else if (cmd == 6) {
          if (msgInfo.type == 6 && msgInfo.content == "刷新好友列表") {
            store.dispatch("loadFriend");
            store.dispatch("loadContact");
          } else {
            store.dispatch("loadGroup");
          }
        }
      });
      wsApi.onClose((res) => {
        // 3000是客户端主动关闭
        if (res.code != 3000) {
          // 重新连接
          uni.showToast({
            title: "连接已断开，尝试重新连接...",
            icon: "none",
          });
          let loginInfo = uni.getStorageSync("zhixin_applets_loginInfo");
          wsApi.reconnect(import.meta.env.VITE_WS_URL, loginInfo.accessToken);
        }
      });
    },
    loadPrivateMessage(minId) {
      store.commit("loadingPrivateMsg", true);
      http({
        url: "/message/private/loadMessage?minId=" + minId,
        method: "GET",
      }).then((msgInfos) => {
        msgInfos.forEach((msgInfo) => {
          msgInfo.selfSend = msgInfo.sendId == store.state.userStore.userInfo.id;
          let friendId = msgInfo.selfSend ? msgInfo.recvId : msgInfo.sendId;
          let friend = store.state.friendStore.friends.find((f) => f.id == friendId);
          if (friend) {
            this.insertPrivateMessage(friend, msgInfo);
          }
        });
        if (msgInfos.length == 100) {
          // 继续拉取
          this.loadPrivateMessage(msgInfos[99].id);
        } else {
          store.commit("loadingPrivateMsg", false);
        }
      });
    },
    loadGroupMessage(minId) {
      store.commit("loadingGroupMsg", true);
      http({
        url: "/message/group/loadMessage?minId=" + minId,
        method: "GET",
      }).then((msgInfos) => {
        msgInfos.forEach((msgInfo) => {
          msgInfo.selfSend = msgInfo.sendId == store.state.userStore.userInfo.userId;
          let groupId = msgInfo.groupId;
          let group = store.state.groupStore.groups.find((g) => g.id == groupId);
          if (group) {
            this.insertGroupMessage(group, msgInfo);
          }
        });
        if (msgInfos.length == 100) {
          // 继续拉取
          this.loadGroupMessage(msgInfos[99].id);
        } else {
          store.commit("loadingGroupMsg", false);
        }
      });
    },
    handlePrivateMessage(msg) {
      // 标记这条消息是不是自己发的
      msg.selfSend = msg.sendId == store.state.userStore.userInfo.userId;
      // 好友id
      let friendId = msg.selfSend ? msg.recvId : msg.sendId;
      // 消息已读处理
      if (msg.type == enums.MESSAGE_TYPE.READED) {
        if (msg.selfSend) {
          // 我已读对方的消息，清空已读数量
          let chatInfo = {
            type: "PRIVATE",
            targetId: friendId,
          };
          store.commit("resetUnreadCount", chatInfo);
        } else {
          // 对方已读我的消息，修改消息状态为已读
          store.commit("readedMessage", friendId);
        }
        return;
      }
      this.loadFriendInfo(friendId).then((friend) => {
        this.insertPrivateMessage(friend, msg);
      });
    },
    insertPrivateMessage(friend, msg) {
      // webrtc 信令
      if (msg.type >= enums.MESSAGE_TYPE.RTC_CALL && msg.type <= enums.MESSAGE_TYPE.RTC_CANDIDATE) {
      }

      let chatInfo = {
        type: "PRIVATE",
        targetId: friend.id,
        showName: friend.nickName,
        secname: friend.secname || friend.firmName,
        headImage: friend.headImage,
      };
      // 打开会话
      store.commit("openChat", chatInfo);
      // 插入消息
      store.commit("insertMessage", msg);
      // 播放提示音
      !msg.selfSend && this.playAudioTip();
    },
    handleGroupMessage(msg) {
      // 标记这条消息是不是自己发的
      msg.selfSend = msg.sendId == store.state.userStore.userInfo.userId;
      let groupId = msg.groupId;
      // 消息已读处理
      if (msg.type == enums.MESSAGE_TYPE.READED) {
        // 我已读对方的消息，清空已读数量
        let chatInfo = {
          type: "GROUP",
          targetId: groupId,
        };
        store.commit("resetUnreadCount", chatInfo);
        return;
      }
      this.loadGroupInfo(groupId).then((group) => {
        // 插入群聊消息
        this.insertGroupMessage(group, msg);
      });
    },
    insertGroupMessage(group, msg) {
      let chatInfo = {
        type: "GROUP",
        targetId: group.id,
        showName: group.remark ? group.remark : group.name,
        headImage: group.headImageThumb,
      };
      // 打开会话
      store.commit("openChat", chatInfo);
      // 插入消息
      store.commit("insertMessage", msg);
      // 播放提示音
      !msg.selfSend && this.playAudioTip();
    },
    loadFriendInfo(id) {
      return new Promise((resolve, reject) => {
        let friend = store.state.friendStore.friends.find((f) => f.id == id);
        if (friend) {
          resolve(friend);
        } else {
          http({
            url: `/friend/find/${id}`,
            method: "get",
          }).then((friend) => {
            store.commit("addFriend", friend);
            resolve(friend);
          });
        }
      });
    },
    loadGroupInfo(id) {
      return new Promise((resolve, reject) => {
        let group = store.state.groupStore.groups.find((g) => g.id == id);
        if (group) {
          resolve(group);
        } else {
          http({
            url: `/im/group/find/${id}`,
            method: "get",
          }).then((group) => {
            resolve(group);
            store.commit("addGroup", group);
          });
        }
      });
    },
    exit() {
      console.log("exit");
      wsApi.close();
      uni.removeStorageSync("zhixin_applets_loginInfo");
      uni.reLaunch({
        url: "/pages/login/index",
      });
      store.dispatch("unload");
    },
    playAudioTip() {
      // 音频播放无法成功
      // this.audioTip = uni.createInnerAudioContext();
      // this.audioTip.src =  "/static/audio/tip.wav";
      // this.audioTip.play();
    },
    initAudit() {
      console.log("initAudit");
      if (store.state.userStore.userInfo.type == 1) {
        // 显示群组功能
        uni.setTabBarItem({
          index: 2,
          text: "群聊",
        });
      } else {
        // 隐藏群组功能
        uni.setTabBarItem({
          index: 2,
          text: "搜索",
        });
      }
    },
    // 设置tabbar 未读消息
    refreshUnreadBadge(unreadCount) {
      if (unreadCount > 0) {
        uni.setTabBarBadge({
          index: 1,
          text: unreadCount + "",
        });
      } else {
        uni.removeTabBarBadge({
          index: 1,
          complete: () => { },
        });
      }
    },
    getTotalUnreadCount() {
      let count = 0;
      store.state.chatStore.chats.forEach((chat) => {
        count += chat.unreadCount;
      });
      return count;
    },
  },
  onLaunch() {
    // 登录状态校验
    if (uni.getStorageSync("zhixin_applets_loginInfo")) {
      // 初始化
      this.init();
    } else {
       let obj = uni.getLaunchOptionsSync()
       console.log(obj);
	   if(obj.path != "pages/invite/invite"){
		 //  跳转到登录页
		   uni.reLaunch({
		     url: "/pages/login/index",
		   });
	   }
      
    }
  },
};
</script>

<style lang="scss">
@import url("./static/icon/iconfont.css");

.tab-page {
  // #ifdef H5
  height: calc(100vh - 46px - 50px); // h5平台100vh是包含了顶部和底部，需要减去
  // #endif
  // #ifndef H5
  height: calc(100vh);
  // #endif
  background-color: #f8f8f8;
}

.page {
  // #ifdef H5
  height: calc(100vh - 45px); // h5平台100vh是包含了顶部，需要减去
  // #endif
  // #ifndef H5
  height: calc(100vh);
  // #endif
  background-color: #f8f8f8;
}
</style>
