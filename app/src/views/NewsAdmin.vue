<!-- ********************************************************************
********************* CHAT ADMIN INTERFACE ******************************
*************************************************************************
This page deals with displaying the chat for the an admin user.
-->

<template>
  <div class="chatroom">
    <div v-if="warning != ''">
      <div class="warning">
      <h4>{{warning}}</h4>
      </div>
    </div>
    <div class="channels">
      <button v-bind:class="{ color : clicked == 'btn1'}" id="channel" @click="getMessagesList('channel1'), changeColor('btn1'), resetCurrentMsgList()">
        <h2>Channel #1</h2>
      </button>
      <button v-bind:class="{ color : clicked == 'btn2'}" id="channel" @click="getMessagesList('channel2'), changeColor('btn2'), resetCurrentMsgList()">
        <h2>Channel #2</h2>
      </button>
    </div>
    <div class="chat">
      <DisplayAllMessages v-bind:messages="messages" v-bind:CurrentMsg="CurrentMsg" />
    </div>
    <div class="sendMessage">
      <div class="center">
        <input
          type="text"
          v-model="myMessage"
          placeholder="  Send message ... "
        />
        <div class="btn">
          <button id="send" @click="sendMsg">SEND</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
// import the component DisplayAllMessages.vue
import DisplayAllMessages from "../components/DisplayAllMessages.vue";

// import axios for the communication with the server
import axios from "axios";

// import config.json
import config from "../../config.json";

export default {
  components: {
    DisplayAllMessages
  },
  // ======================= MOUNTED ==============================
  // ===== as the page is rendered execute these functions ========
  mounted() { 
    this.isOffline()                    // set the warning on the interface if the app is offline
    this.isOfflineAsync()               // set the warning for offline use if the connection is lost
    this.isOnlineAsync()                // set the warning for online use if the connection is back
    this.getMessagesList("channel1")    // load messages for 'channel1' as the page is opened
    this.changeColor('btn1')            // highlight 'channel1' button
  },
  // ======================== DATA ================================
  data() {
    return {
      CurrentMsg: [],                 // stores the new message(s) the admin will write
      clicked: '',                    // keep track which button has been clicked on
      messages: [],                   // stores the previous messages written by all the admins in a specific channel
      SelectedChannel: "channel1",    // keeps track of the selected channel
      myMessage: "",                  // stores the messages written in the input field
      username: "Admin",              // stores the username of the Admin connected to the webSocket ( connection is done in the App.vue, hence as the app is launched )
      isConnected:navigator.onLine,   // keeps track if a internet connection exists. It is initalize to false/true as the app is opened.
      warning:''                      // Variable to prin a warning on the interface
   }
  },
  // ========================= SOCKET =============================
  // ================== WebSocket management ======================
  sockets: {
    // Fired when the server sends something on the "messageChannel" channel.
    newMessage(data) {
      if (this.SelectedChannel == data.channel) {
        this.CurrentMsg.push(data);
      }
    },
  },
  // ========================= METHOD =============================
  methods: {
    // ***************************** isOfflineAsync ********************************
    // Async function listener to the 'offline' event of the system. 
    // As the connection is lost it sets the warning to be printed on the interface
    // and lets the app know that it is offline
    // *****************************************************************************
    isOfflineAsync: function() {
        window.addEventListener('offline', ()=> {
        console.log("You lost connection.")
        this.isConnected = false
        this.warning = "Connection lost. Cannot send/receive messages."
      })
    },

    // ***************************** isOffline ********************************
    // Sync function for setting the warning if the system is launched offline.
    // When the app is opened offline, this function sets soon the warning
    // *****************************************************************************
    isOffline: function() {
      if (!this.isConnected){
        this.warning = "Connection lost. Cannot send/receive messages."
      }
    },
    // ************************************ isOnlineAsync *************************************
    // Async function listener to the 'online' event of the system. 
    // As the connection is established again, it 'removes' the warning and lets the app know that it is online
    // ****************************************************************************************
    isOnlineAsync: function() {
        window.addEventListener('online', ()=> {
        console.log("You are now back online.");
        this.isConnected = true
        this.warning = ""
      })
    },
    // ************************* ResetCurretMsgList ******************************
    // It reinializes the CurrentMsg[] for storing the new current messages
    // ***************************************************************************
    resetCurrentMsgList(){
      this.CurrentMsg = []
    },

    // ************************* ChangeColor ******************************
    // Changes the background color of the btn as they are clicked
    // ********************************************************************
    changeColor(btn){
      this.clicked = btn;
    },

    // ***************************** sendMsg ******************************
    // Stores the current msg in CurrentMsg and emits it on the websocket
    // ********************************************************************
    sendMsg() {
      
      if (this.isConnected) {
        const now = new Date();

        // to set the date in french, set 'fr-FR' instead of 'en-GB'
        const thisMoment = now.toLocaleString("en-GB", {
          year: "numeric",
          month: "long",
          day: "numeric",
          hour: "2-digit",
          minute: "2-digit",
        });

        const data = {
          user: this.username,
          content: this.myMessage,
          channel: this.SelectedChannel,
          date: thisMoment,
          FullDate: now
        };

        this.CurrentMsg.push(data);

        // this.$socket.client is `socket.io-client` instance
        this.$socket.client.emit("newMessage", data);
      }
    },
    
    // ***************************** setChannel ******************************
    // Stores the selected channel on SelectedChannel
    // ***********************************************************************
    setChannel(channel) {
      this.SelectedChannel = channel;
    },

    // ********************************* getMessagesList **********************************
    // Retrieve from the server the list of the last 100 messages to show on the interface
    // ************************************************************************************
    getMessagesList(channel) {

      // Set the Selected channel
      this.setChannel(channel);

      // temp variable to store the last msgs
      var oldMessages;

      if (this.isConnected) {    // if we are connected to internet

        // ========================= START SERVER REQUEST ==============================

        // set the address for fetching the previous messages
        const resourse = config.URL.domain + config.URL.getMessages + this.SelectedChannel;

        axios
          // send request to the server
          .get(resourse)

          .then((response) => {

            // fetch the JSON data sent back by the server
            oldMessages = response.data;

            this.messages = oldMessages;

            // stores it on the local storage
            localStorage.setItem(
              this.SelectedChannel,
              JSON.stringify(oldMessages)
            );

            console.log("Messages retrieved from server");
          })
          .catch(function (error) {
            // handle error
            console.log(error);
          });
        // ========================= END SERVER REQUEST ==============================
      } 
      
      else {   // if we are offline
        
        // get last messages from the local storage
        oldMessages = localStorage.getItem(this.SelectedChannel);

        console.log("messages retrieve from localStorage");
        
        // USE CASE: if app is opened offline and no previous messages are stored on local storage, oldMessages will be NULL
        if (oldMessages != null) {
          // convert the string in JSON
          this.messages = JSON.parse(oldMessages);
        }
      }
    },
  },
};
</script>

<style scoped>
.warning {
    color: rgba(207, 41, 41, 0.986);
}
.color {
  background-color: rgba(0, 0, 0, 0.062);
}
.chatroom {
  background-color: #f0f0f0;
  -webkit-background-clip: padding-box;
  background-clip: padding-box;
  border: 1px solid rgba(0, 0, 0, 0.2);
  border-radius: 3px;
  outline: 0;
  -webkit-box-shadow: 0 3px 9px rgba(0, 0, 0, 0.5);
  box-shadow: 0 3px 9px rgba(0, 0, 0, 0.5);
  margin: auto;
  width: 70%;
  min-height: 400px;
  border: 3px solid rgb(0, 0, 0);
  padding: 10px;
  text-align: center;
}

#channel {
  width: 100%;
  border: none;
  border-bottom: 3px solid rgb(0, 0, 0);
}

button:hover {
  color: rgb(117, 108, 108);
  cursor: pointer;
}

button:active {
  background-color: rgba(0, 0, 0, 0.062);
  outline: none;
  border: none;
}

.channels {
  width: 20%;
  float: left;
}

.chat {
  min-height: 350px;
  margin-left: 20%;
  background: rgba(0, 0, 0, 0.062);
}

.sendMessage {
  min-height: 50px;
  background: rgba(53, 46, 46, 0.253);
  margin-left: 20%;
}

input {
  width: 80%;
  height: 40px;
  margin-top: 2px;
  border-radius: 3px;
  float: left;
}

input:focus {
  outline: none;
  border-radius: 3px;
}

.btn {
  margin-left: 81%;
  border: none;
}

.btn:active {
  border: none;
}

#send {
  width: 95%;
  height: 40px;
  border-radius: 3px;
  margin-top: 6px;
}
</style>