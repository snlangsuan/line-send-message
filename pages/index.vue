<template>
  <v-container fluid fill-height>
    <v-layout row wrap>
      <v-flex pt-3 pr-3 xs6>
        <div>
          <v-text-field
            v-model="token"
            :error-messages="errors.token"
            label="LINE channel access token"
          ></v-text-field>
          <v-combobox
            v-model="userIds"
            :error-messages="errors.userIds"
            chips
            multiple
            label="ID of the target recipient"
          >
            <template v-slot:selection="data">
              <v-chip
                :selected="data.selected"
                close
                @input="removeUser(data.item)"
              >
                <strong>{{ data.item }}</strong>&nbsp;
              </v-chip>
            </template>
          </v-combobox>
          <div style="width: 100%;display: flex;align-items: center;" :class="{'error-messages': !!errors.messages || errors.messages.length > 0}">
            <span class="title">
              <v-icon medium>chat_bubble</v-icon>
              Messages
            </span>
            <v-spacer></v-spacer>
            <v-btn flat icon class="mr-0">
              <v-icon :disabled="!canAdd" @click="addMessage">add</v-icon>
            </v-btn>
          </div>
          <v-list style="width: 100%;background: transparent !important;">
            <div v-if="messages.length < 1 && errors.messages" class="error--text text-xs-center">{{ errors.messages }}</div>
            <template v-for="(msg, index) in messages">
              <v-list-tile
                :key="msg.id"
                avatar
                :class="{ 'list-seleted': msg.selected }"
                @click="selectMessage(msg.id)"
              >
                <v-list-tile-avatar>
                  <v-icon>{{ msg.icon }}</v-icon>
                </v-list-tile-avatar>
                <v-list-tile-content>
                  <v-list-tile-title v-text="msg.name"></v-list-tile-title>
                </v-list-tile-content>
                <v-list-tile-action>
                  <v-icon small @click="removeMessage(msg.id)">delete</v-icon>
                </v-list-tile-action>
              </v-list-tile>
              <v-divider
                v-if="index + 1 < messages.length"
                :key="index"
              ></v-divider>
            </template>
          </v-list>
        </div>
      </v-flex>
      <v-flex ref="app_codes" pl-3 xs6>
        <textarea v-if="selected" v-model="selected.message" class="app-codes"></textarea>
      </v-flex>
    </v-layout>
    <div ref="app_actions" class="app-actions">
      <v-btn color="primary" @click="sendMessage">Send</v-btn>
      <v-btn>Clear</v-btn>
    </div>
  </v-container>
</template>

<script>
export default {
  data() {
    return {
      token: '',
      userIds: [],
      messages: [],
      selected: null,
      errors: {
        token: '',
        userIds: '',
        messages: ''
      }
    }
  },
  computed: {
    canAdd() {
      const correct = this.messageCanUse()
      return this.messages.length < 5 && correct.length === this.messages.length
    }
  },
  watch: {
    selected: {
      handler(val) {
        if (!val) return
        try {
          const data = JSON.parse(val.message)
          this.selected.name = data.type || ''
        } catch(error) {}
      },
      deep: true
    },
    messages(val) {
      if (val.length > 0)
        this.errors.messages = ''
    }
  },
  mounted() {
    const action = this.$refs.app_actions
    this.$refs.app_codes.style.marginBottom = action.clientHeight + 'px'
  },
  methods: {
    removeUser (item) {
      this.userIds.splice(this.userIds.indexOf(item), 1)
      this.userIds = [...this.userIds]
    },
    messageCanUse() {
      return this.messages.reduce((list, x) => {
        try {
          list.push(JSON.parse(x.message))
        } catch(e) {}
        return list
      }, [])
    },
    addMessage() {
      if (this.messages.length > 5) return
      this.clearSelected()
      this.messages.push({
        id: Date.now(),
        message: '',
        name: '',
        icon: 'format_size',
        selected: true
      })
      this.selected = this.messages[this.messages.length - 1]
    },
    removeMessage(id) {
      let idx = this.messages.findIndex(x => x.id === id)
      this.messages.splice(idx, 1)
    },
    clearSelected() {
      let item = this.messages.find(x => x.selected)
      if (!item) return
      item.selected = false
    },
    selectMessage(id) {
      this.clearSelected()
      let item = this.messages.find(x => x.id === id)
      if (!item) return
      item.selected = true
      this.selected = item
    },
    sendMessage() {
      if (!this.token) 
        this.errors.token = 'LINE channel access token is required'

      console.log(this.userIds);
      if (!this.userIds || this.userIds.length < 1)
        this.errors.userIds = 'User ID not found'

      if (this.messageCanUse() < 1)
        this.messages = []

      if (!this.messages || this.messages.length < 1)
        this.errors.messages = 'Messages not found'

      if (this.token && this.userIds.length > 0 && this.messages.length > 0) {
        this.$axios({
          method: 'post',
          url: 'https://api.line.me/v2/bot/message/push',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.token}`
          },
          data: {
            to: this.userIds,
            messages: this.messages
          }
        })
        .then(response => {
          console.log(response)
        })
        .catch(error => {
          console.error(error)
        })
      }
    }
  },
  head() {
    return {
      title: 'Test Send LINE Message'
    }
  }
}
</script>

<style>
.app-actions {
  width: 100%;
  position: fixed;
  bottom: 0;
  left: 0;
  padding: 8px;
  text-align: center;
  background-color: #ffffff;
}
.app-codes {
  width: 100%;
  height: 100%;
  padding: 8px;
  outline: none;
}
.list-seleted {
  background-color: #2cb900;
}
.list-seleted .v-icon,
.list-seleted .v-list__tile__title {
  color: #ffffff !important;
}
.error-messages .v-icon,
.error-messages span {
  color: #dd2c00 !important;
}
</style>
