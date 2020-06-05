<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <div class="room">
      <div>
        自分を画面に表示する：<input type="checkbox" v-model="selfDisp" /><br />
        マイクのオン／オフを切り替える：<input type="checkbox" v-model="microphoneFlg" @change="changeMicrophone" /><br />
        画面のオン／オフを切り替える：<input type="checkbox" v-model="videoFlg" @change="changeVideo" /><br />
        <video v-show="selfDisp" ref="jsLocalStream" id="jsLocalStream" muted playsinline></video>
        <div>
          <p>任意のルーム名を入力してください。</p>
          <p class="errMsg" v-if="errors.length">{{errors[0]}}</p>
          ルーム名：<input type="text" placeholder="Room Name" id="jsRoomId" v-model="jsRoomId" v-bind:disabled="joinFlg" />
        </div>
        <button id="jsJoinTrigger" @click="join" v-bind:disabled="joinFlg">入室</button>
        <button ref="jsLeaveTrigger" id="jsLeaveTrigger" v-bind:disabled="!joinFlg">退出</button>
      </div>

      <div ref="jsRemoteStreams" class="remoteStreams" id="jsRemoteStreams"></div>

      <div>
        <div id="messages">
          <div v-for="(item,i) in messageList" v-bind:key="i" v-bind:style="{textAlign: item.align}">{{item.message}}</div>
        </div>
        <div>
          <input type="text" id="jsLocalText" v-model="jsLocalText" v-bind:disabled="!joinFlg" />
        </div>
        <button ref="jsSendTrigger" id="jsSendTrigger" v-bind:disabled="!joinFlg">メッセージ送信</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data(){
    return {
      jsRoomId: "",
      localStream: {},
      jsLocalText: "",
      errors: [],
      joinFlg: false,
      messageList: [],
      selfDisp: false,
      microphoneFlg: true,
      videoFlg: true
    }
  },
  methods: {
    join: function(){
      let self = this;
      
      // 入力チェック
      if(self.jsRoomId.trim() === ''){
        self.errors.push('ルーム名は必須です。');
        self.jsRoomId = '';
        return;
      }
      
      // ピアに接続できなければエラーを入室させない
      if (!peer.open) {
        alert('接続を確立することができませんでした。再度お試しください。');
        return;
      }

      // メッシュ方式でルームを作成する
      const room = peer.joinRoom(self.jsRoomId, {
        mode: 'mesh',
        stream: self.localStream,
      });

      // ルーム関連のイベントリスナーを設定
      room.once('open', () => {
        // MSG1: You joined
        self.messageList.push({
          message: '=== '+ peer.id +' You joined ===\n',
          align: 'left'
        });
        self.joinFlg = true;
      });
      room.on('peerJoin', peerId => {
        // MSG2: who joined
        self.messageList.push({
          message: `=== ${peerId} joined ===\n`,
          align: 'right'
        });
      });

      // Render remote stream for new peer join in the room
      room.on('stream', async stream => {
        const newVideo = document.createElement('video');
        newVideo.srcObject = stream;
        newVideo.playsInline = true;
        // mark peerId to find it later at peerLeave event
        newVideo.setAttribute('data-peer-id', stream.peerId);
        self.$refs.jsRemoteStreams.append(newVideo);
        await newVideo.play().catch(console.error);
      });

      // メッセージの設定
      room.on('data', ({ data, src }) => {
        // MSG3: who message sent
        self.messageList.push({
          message: `${src}: ${data}\n`,
          align: 'right'
        });
      });

      // ルームメイトが退室した時の処理
      room.on('peerLeave', peerId => {
        const remoteVideo = self.$refs.jsRemoteStreams.querySelector(
          `[data-peer-id=${peerId}]`
        );
        remoteVideo.srcObject.getTracks().forEach(track => track.stop());
        remoteVideo.srcObject = null;
        remoteVideo.remove();

        // MSG4: who left room
        self.messageList.push({
          message: `=== ${peerId} left ===\n`,
          align: 'right'
        });
      });

      // 自分が退室した時の処理
      room.once('close', () => {
        self.$refs.jsSendTrigger.removeEventListener('click', onClickSend);

        // MSG5: you left room
        self.messageList.push({
          message: '=== '+ peer.id +' You left ===\n',
          align: 'left'
        });
        Array.from(self.$refs.jsRemoteStreams.children).forEach(remoteVideo => {
          remoteVideo.srcObject.getTracks().forEach(track => track.stop());
          remoteVideo.srcObject = null;
          remoteVideo.remove();
        });
        self.joinFlg = false;
      });

      // ボタンイベントを設定
      self.$refs.jsSendTrigger.addEventListener('click', onClickSend);
      self.$refs.jsLeaveTrigger.addEventListener('click', () => room.close(), { once: true });

      // メッセージ送信時の処理
      function onClickSend() {
        if (self.jsLocalText.trim().length > 50){
          alert('メッセージは50文字までにしてください。');
          return;
        }
        // Send message to all of the peers in the room via websocket
        if(self.jsLocalText.trim() !== ''){
          room.send(self.jsLocalText.trim());
          // MSG6: your free message sent
          self.messageList.push({
            message: `${peer.id}: ${self.jsLocalText.trim()}\n`,
            align: 'left'
          });
          self.jsLocalText = '';
        }
      }
    },
    changeMicrophone: function(){
      this.localStream.getAudioTracks()[0].enabled = this.microphoneFlg;
    },
    changeVideo: function(){
      this.localStream.getVideoTracks()[0].enabled = this.videoFlg;
    }
  },
  mounted: function(){
    let self = this;
    (async function main() {
      const localVideo = self.$refs.jsLocalStream;
      self.localStream = await navigator.mediaDevices
        .getUserMedia({
          audio: true,
          video: true,
        })
        .catch(console.error);

      // Render local stream
      localVideo.srcObject = self.localStream;
      await localVideo.play().catch(console.error);
      peer.on('error', console.error);
    })();
  }
}

// import
import Peer from 'skyway-js';
// eslint-disable-next-line require-atomic-updates
const peer =  new Peer({
  key: '59093eb6-f094-476c-af62-ab068434ec23'
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
.errMsg {
  color: red;
}
#messages {
  padding: 10px 100px;
}
</style>
