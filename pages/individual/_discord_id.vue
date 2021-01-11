<template>
  <div>
    <client-only>
      <template v-if="config.token">
        <div class="grid" :class="{ bounce: config.bounce }" :style="gridStyle">
          <div class="member" :class="{ speaking: member.speaking }" :style="memberStyle"></div>
        </div>
      </template>
    </client-only>
  </div>
</template>

<script>
import jwt_decode from 'jwt-decode'

export default {
  layout: 'empty',
  data() {
    return {
      members: new Map(),
      avatars: new Map(),
      pendingAvatars: new Set(),

      // Hack around lack of reactivity for Map & Set
      memberStateCounter: 1,
      avatarStateCounter: 1,

      // Config updating
      configSocket: null,

      // RPC stuff
      socket: null,
      channelID: null,
      connectionTries: 0,
      commandCounter: 0,
      pendingCommands: {},

      config: {},
    }
  },
  computed: {
    gridStyle() {
      return {}
    },
    memberStyle() {
      return {
        'background-image': `url(${this.member.image})`,
      }
    },
    member() {
      return (
        (this.memberStateCounter &&
          this.avatarStateCounter &&
          Array.from(this.members.values())
            .filter((m) => this.avatars.has(m.id))
            .map((m) => ({
              ...m,
              image: this.avatars.get(m.id)
                ? `https://cdn.discord-reactive-images.fugi.tech/${this.avatars.get(m.id)}`
                : `https://cdn.discordapp.com/avatars/${m.id}/${m.avatar}.png?size=1024`,
            }))
            .find((m) => m.id === this.$route.params.discord_id)) ||
        {}
      )
    },
  },
  watch: {
    memberStateCounter() {
      this.members.forEach(async (m) => {
        if (this.avatars.has(m.id) || this.pendingAvatars.has(m.id)) return
        this.pendingAvatars.add(m.id)

        if (this.configSocket && this.configSocket.readyState === 1) {
          this.configSocket.send(JSON.stringify({ avatar: m.id }))
        }
      })
    },
    config(config) {
      localStorage.setItem('config', JSON.stringify(config))
    },
    'config.jwt'(jwt, oldJWT) {
      console.log(arguments)
      if (jwt === oldJWT) return

      if (this.configSocket) {
        this.configSocket.onclose = null
        this.configSocket.close()
      }

      const connect = () => {
        this.configSocket = new WebSocket(`${location.protocol === 'https:' ? 'wss' : 'ws'}://${location.host}/api`)
        this.configSocket.onopen = () => {
          this.configSocket.send(JSON.stringify({ jwt }))
          for (const [id, _] of this.avatars) {
            this.configSocket.send(JSON.stringify({ avatar: id }))
          }
          for (const id of this.pendingAvatars) {
            this.configSocket.send(JSON.stringify({ avatar: id }))
          }
        }
        this.configSocket.onmessage = (event) => {
          const d = JSON.parse(event.data)
          this.config = Object.assign({}, this.config, d.config)
          for (const [id, avatar] of Object.entries(d.avatars || {})) {
            this.avatars.set(id, avatar)
            this.pendingAvatars.delete(id)
          }
          this.memberStateCounter++
        }
        this.configSocket.onclose = (err) => {
          console.error('Config WS failed:', err)
          this.configSocket = null
          setTimeout(connect, 1000)
        }
      }

      connect()
    },
  },
  mounted() {
    try {
      this.config = JSON.parse(localStorage.getItem('config')) || {}
    } catch (_) {}

    if (this.config.jwt) {
      const now = 600 + new Date() / 1000
      const jwt = jwt_decode(this.config.jwt)
      if (now >= jwt.exp) {
        localStorage.removeItem('config')
        location.reload()
        return
      }
    }

    this.connect()
  },
  beforeDestroy() {
    if (this.socket) {
      this.socket.onclose = null
      this.socket.close()
      this.socket = null
    }
  },
  methods: {
    async subscribe(channelID) {
      // Do nothing if we're already on that channel
      if (channelID === this.channelID) return

      // If we were on an old channel then unsubscribe (without blocking)
      if (this.channelID) {
        this.request('UNSUBSCRIBE', { channel_id: this.channelID }, 'VOICE_STATE_CREATE')
        this.request('UNSUBSCRIBE', { channel_id: this.channelID }, 'VOICE_STATE_UPDATE')
        this.request('UNSUBSCRIBE', { channel_id: this.channelID }, 'VOICE_STATE_DELETE')
        this.request('UNSUBSCRIBE', { channel_id: this.channelID }, 'SPEAKING_START')
        this.request('UNSUBSCRIBE', { channel_id: this.channelID }, 'SPEAKING_STOP')
      }

      // Update state
      this.channelID = channelID
      this.members.clear()
      this.memberStateCounter++

      if (!channelID) return

      // Subscribe to what we need (without blocking)
      this.request('SUBSCRIBE', { channel_id: channelID }, 'VOICE_STATE_CREATE')
      this.request('SUBSCRIBE', { channel_id: channelID }, 'VOICE_STATE_UPDATE')
      this.request('SUBSCRIBE', { channel_id: channelID }, 'VOICE_STATE_DELETE')
      this.request('SUBSCRIBE', { channel_id: channelID }, 'SPEAKING_START')
      this.request('SUBSCRIBE', { channel_id: channelID }, 'SPEAKING_STOP')

      // Get list of members in channel (blocking)
      const channel = await this.request('GET_CHANNEL', { channel_id: channelID })
      if (channelID !== this.channelID) return

      for (const v of channel.voice_states) {
        this.members.set(v.user.id, {
          id: v.user.id,
          avatar: v.user.avatar,
          name: v.nick,
          speaking: false,
        })
      }
      this.memberStateCounter++
    },

    async evt_READY() {
      if (!this.config.token) {
        const d = await this.request('AUTHORIZE', {
          client_id: '794365445557846066',
          scopes: ['rpc', 'identify'],
          prompt: 'none',
        })
        console.log('authorize', d)
        this.config = await this.$api.code(d.code)
      }
      try {
        await this.request('AUTHENTICATE', { access_token: this.config.token })
      } catch (_) {
        localStorage.removeItem('config')
        location.reload()
        return
      }
      await this.request('SUBSCRIBE', {}, 'VOICE_CHANNEL_SELECT')
      const channel = await this.request('GET_SELECTED_VOICE_CHANNEL')
      if (channel) this.subscribe(channel.id)
    },

    async evt_VOICE_CHANNEL_SELECT({ channel_id }) {
      this.subscribe(channel_id)
    },

    async evt_VOICE_STATE_CREATE(v) {
      this.members.set(v.user.id, {
        id: v.user.id,
        avatar: v.user.avatar,
        name: v.nick,
        speaking: false,
      })
      this.memberStateCounter++
    },

    async evt_VOICE_STATE_UPDATE(v) {
      const m = this.members.get(v.user.id)
      if (m) {
        m.avatar = v.user.avatar
        m.name = v.nick
        this.memberStateCounter++
      }
    },

    async evt_VOICE_STATE_DELETE(v) {
      this.members.delete(v.user.id)
      this.memberStateCounter++
    },

    async evt_SPEAKING_START({ user_id }) {
      const m = this.members.get(user_id)
      if (m) {
        m.speaking = true
        this.memberStateCounter++
      }
    },

    async evt_SPEAKING_STOP({ user_id }) {
      const m = this.members.get(user_id)
      if (m) {
        m.speaking = false
        this.memberStateCounter++
      }
    },

    connect(tries = 0) {
      if (this.socket) return

      try {
        const port = 6463 + (tries % 10)
        this.socket = new WebSocket(`ws://127.0.0.1:${port}/?v=1&client_id=794365445557846066`)
      } catch (_) {
        this._handleClose({ code: 1006 })
        return
      }

      this.socket.onclose = this._handleClose.bind(this)
      this.socket.onmessage = this._handleMessage.bind(this)
    },

    /* async */ request(cmd, args, evt) {
      if (!this.socket) return

      const nonce = `counter:${++this.commandCounter}`

      const p = new Promise((resolve, reject) => {
        this.pendingCommands[nonce] = { resolve, reject }
      })

      this.socket.send(JSON.stringify({ cmd, args, evt, nonce }))

      return p
    },

    _handleClose(e) {
      console.error('WS Closed: ', e)

      try {
        this.socket.close()
      } catch (e) {}

      this.socket = null

      const tries = e.code === 1006 ? ++this.connectionTries : 0
      const backoff = Math.pow(2, Math.floor(tries / 10))
      setTimeout(() => this.connect(tries), backoff)
    },

    _handleMessage(message) {
      let payload = null

      try {
        payload = JSON.parse(message.data)
      } catch (e) {
        console.error('Payload not JSON: ', payload)
        return
      }

      let { cmd, evt, nonce, data } = payload

      // console.log('Incoming Payload: ', payload)

      if (cmd === 'DISPATCH') {
        const method = `evt_${evt}`
        if (this[method]) {
          this[method](data)
        }
        return
      }

      if (!this.pendingCommands[nonce]) return
      const { resolve, reject } = this.pendingCommands[nonce]
      delete this.pendingCommands[nonce]

      if (evt === 'ERROR') {
        const error = new Error(data.message)
        error.code = data.code
        reject(error)
      } else {
        resolve(data)
      }
    },

    evt_ERROR(data) {
      console.error('Dispatched Error: ', data)
      this.socket.close()
    },
  },
}
</script>

<style>
body {
  color: white;
  background: black;
}

.grid {
  display: grid;
  height: 100vh;
  overflow: hidden;
}

.member {
  background-size: contain;
  background-position: top center;
  filter: brightness(50%);
  transition: filter 200ms linear;
}

.member.speaking {
  filter: brightness(100%);
}

.bounce .member {
  position: relative;
  top: 10px;
}

.bounce .member.speaking {
  animation: 200ms bounce;
}

.member img {
  width: 100%;
}

@keyframes bounce {
  0% {
    top: 10px;
  }
  50% {
    top: 0px;
  }
  100% {
    top: 10px;
  }
}
</style>
