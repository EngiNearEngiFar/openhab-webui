<template>
  <div class="main-container">
    <!-- Show settings gear for local settings if intercom is enabled and in edit mode -->
    <f7-button v-if="context.editmode" icon-f7="gear_fill" @click.stop="localSettingsPopup()" class="text-align-right">
      Local settings
    </f7-button>
    <div v-if="config.enableVideo" class="video-container" :style="{ 'aspect-ratio': config.defaultVideoAspectRatio || '4/3' }">
      <video ref="remoteVideo" autoplay playsinline class="remote-video" poster="@/images/openhab-logo.svg" />
      <!-- Conditionally show local video when ready to prevent ugly video placeholder on Android -->
      <video v-show="showLocalVideo" ref="localVideo" autoplay playsinline muted="muted" class="local-video" />
    </div>
    <!-- Show yellow dial button if connection is not established -->
    <f7-button v-if="!connected" :style="{ height: config.iconSize + 'px' }" icon-f7="phone_fill_arrow_up_right" icon-color="yellow" :icon-size="config.iconSize" />
    <!-- Show dial menu when there`s no call -->
    <f7-button v-else-if="(!session || session.isEnded())" :style="{ height: config.iconSize + 'px' }" icon-f7="phone_fill_arrow_up_right" icon-color="green" :icon-size="config.iconSize" @click.stop="dial()" />
    <!-- Show answer button on incoming call -->
    <f7-segmented v-else-if="session && session.direction === 'incoming' && session.isInProgress()">
      <f7-button :style="{ height: config.iconSize + 'px' }" icon-f7="phone_fill_arrow_down_left" icon-color="green" :icon-size="config.iconSize" @click.stop="answer()">
        {{ (!config.hideCallerId) ? this.remoteParty : '' }}
      </f7-button>
      <f7-button :style="{ height: config.iconSize + 'px' }" icon-f7="phone_down_fill" icon-color="red" :icon-size="config.iconSize" @click.stop="session.terminate()" />
    </f7-segmented>
    <f7-segmented v-else>
      <!-- Show hangup button for outgoing call -->
      <f7-button v-if="session && session.isInProgress()" :style="{ height: config.iconSize + 'px' }" icon-f7="phone_down_fill" icon-color="yellow" :icon-size="config.iconSize" @click.stop="session.terminate()" />
      <!-- Show hangup button for ongoing call -->
      <f7-button v-else-if="session && !session.isEnded()" :style="{ height: config.iconSize + 'px' }" icon-f7="phone_down_fill" icon-color="red" :icon-size="config.iconSize" @click.stop="session.terminate()" />
      <!-- Show send dtmf button if in a call and feature is enabled-->
      <f7-button v-if="session && !session.isInProgress() && !session.isEnded() && config.dtmfString && config.dtmfString.length > 0" :style="{ height: config.iconSize + 'px' }" icon-f7="number_square" icon-color="orange" :icon-size="config.iconSize" @click.stop="sendDTMF()" />
    </f7-segmented>
  </div>
</template>

<style lang="stylus">
.main-container
  width: 100%
  .video-container
    position relative
    .remote-video
      width: 100%
      height: 100%
    .local-video
      position absolute
      width 30%
      bottom 0px
      right 0px
</style>

<script>
import mixin from '../widget-mixin'
import { OhSIPClientDefinition } from '@/assets/definitions/widgets/system'
import foregroundService from '../widget-foreground-service'
import { actionsMixin } from '../widget-actions'
import WidgetConfigPopup from '@/components/pagedesigner/widget-config-popup.vue'
import { WidgetDefinition, pg, pt } from '@/assets/definitions/widgets/helpers.js'

// Thanks to Joseph Sardin, https://bigsoundbank.com
// ringFile source: https://bigsoundbank.com/detail-0375-phone-ring-5.html
import ringFile from './oh-sipclient-ringtone.mp3'
import ringBackFile from './oh-sipclient-ringback.mp3'

export default {
  data () {
    return {
      connected: false,
      session: false,
      remoteParty: '',
      phonebook: new Map(),
      loggerPrefix: 'oh-sipclient',
      showLocalVideo: false
    }
  },
  mixins: [mixin, foregroundService, actionsMixin],
  widget: OhSIPClientDefinition,
  methods: {
    startForegroundActivity () {
      // Load device specific configuration
      this.localConfig = JSON.parse(localStorage.getItem('openhab.ui:sipConfig'))
      // Init phonebook Map
      if (this.config.phonebook) {
        if (this.config.phonebook.includes('=')) {
          this.config.phonebook.split(',').map((e) => {
            return this.phonebook.set(e.split('=')[0], e.split('=')[1])
          })
        }
      }

      if (this.context.editmode) return // do not connect SIP while editing

      // Make sure we have Mic/Camera permissions
      if (!navigator.mediaDevices) {
        this.$f7.dialog.alert('Please ensure that HTTPS is in use and WebRTC is supported in this browser.')
      } else {
        navigator.mediaDevices.getUserMedia({ audio: true, video: this.config.enableVideo })
          .then((stream) => {
            // Start SIP connection
            this.sipStart()
          })
          .catch((err) => {
            console.log('could not access microphone/camera', err)
            this.$f7.dialog.alert('To use the SIP widget you must allow microphone/camera access in your browser and reload this page.')
          })
      }
    },
    stopForegroundActivity () {
      if (this.phone) this.phone.stop()
    },
    sipStart () {
      if (this.phone) this.phone.stop() // reconnect to reload config
      this.context.component.config = { ...this.config, ...this.localConfig } // merge local device configuration

      import(/* webpackChunkName: "jssip" */ 'jssip').then((JsSIP) => { // lazy load jssip
        this.config.enableSIPDebug ? JsSIP.debug.enable('JsSIP:*') : JsSIP.debug.disable()
        // SIP user agent setup
        this.remoteAudio = new window.Audio()
        const url = new URL(this.config.websocketUrl, window.location.origin)
        if (url.protocol.indexOf('http') === 0) {
          url.protocol = url.protocol.replace('http', 'ws')
        }
        const socket = new JsSIP.WebSocketInterface(url.toString())
        const configuration = {
          sockets: [socket],
          uri: 'sip:' + this.config.username + '@' + this.config.domain,
          password: this.config.password,
          session_timers: false
        }
        this.phone = new JsSIP.UA(configuration)

        // Update connected status on connection changes
        this.phone.on('connected', () => {
          this.connected = true
          console.info(this.loggerPrefix + ': Connected to SIP server')
        })
        this.phone.on('disconnected', () => {
          this.connected = false
          console.info(this.loggerPrefix + ': Disconnected from SIP server')
        })

        // Register event for new incoming or outgoing call event
        this.phone.on('newRTCSession', (data) => {
          this.session = data.session

          this.remoteParty = (this.phonebook.size > 0) ? this.phonebook.get(this.session.remote_identity.uri.user) : this.session.remote_identity.uri.user

          if (this.session.direction === 'outgoing') {
            // Handle accepted call
            this.session.on('accepted', () => {
              this.stopTones()
              console.info(this.loggerPrefix + ': Outgoing call in progress')
            })
          } else if (this.session.direction === 'incoming') {
            console.info(this.loggerPrefix + ': Incoming call from ' + this.remoteParty)
            this.playTone(ringFile)
            // Handle accepted call
            this.session.on('accepted', () => {
              console.info(this.loggerPrefix + ': Incoming call in progress')
            })
          }
          // Handle ended call
          this.session.on('ended', () => {
            this.stopMedia()
            console.info(this.loggerPrefix + ': Call ended')
          })
          // Handle failed call
          this.session.on('failed', (event) => {
            this.stopTones()
            this.stopMedia()
            console.info(this.loggerPrefix + ': Call failed. Reason: ' + event.cause)
          })
        })
        this.phone.start()
      })
    },
    playTone (file) {
      if (this.config.enableTones === true) {
        console.info(this.loggerPrefix + ': Starting to play tone')
        this.audio = new Audio(file)
        // Play tone
        this.audio.loop = true
        this.audio.load()
        this.audio.play().catch(error => {
          console.debug(this.loggerPrefix + ': Play tone: ' + error)
        })
      }
    },
    stopTones () {
      if (this.config.enableTones === true) {
        console.info(this.loggerPrefix + ': Stop playing tone')
        this.audio.pause()
      }
    },
    attachMedia () {
      this.session.connection.addEventListener('track', (track) => {
        if (this.config.enableVideo) {
          this.$refs.remoteVideo.srcObject = track.streams[0]
          if (this.config.enableLocalVideo) {
            this.showLocalVideo = true
            navigator.mediaDevices.getUserMedia({ audio: true, video: true })
              .then((stream) => {
                this.$refs.localVideo.srcObject = stream
              })
          }
        } else {
          this.remoteAudio.srcObject = track.streams[0]
          this.remoteAudio.play()
        }
      })
    },
    stopMedia () {
      if (this.config.enableVideo) this.$refs.remoteVideo.srcObject = null
      if (this.config.enableLocalVideo) {
        this.$refs.localVideo.srcObject = null
        this.showLocalVideo = false
      }
    },
    call (target) {
      console.info(this.loggerPrefix + ': Calling ' + this.remoteParty + ' ...')
      this.phone.call(target, { mediaConstraints: { audio: true, video: this.config.enableVideo } })
      this.attachMedia()
      this.playTone(ringBackFile)
    },
    answer () {
      this.stopTones()
      this.session.answer({ mediaConstraints: { audio: true, video: this.config.enableVideo } })
      this.attachMedia()
    },
    sendDTMF () {
      const options = {
        'duration': 160,
        'interToneGap': 640
      }
      this.session.sendDTMF(this.config.dtmfString, options)
    },
    localSettingsPopup () {
      console.info(this.loggerPrefix + ': Opening local settings popup.')
      const popup = { component: WidgetConfigPopup }
      this.$f7router.navigate({ url: 'local-sip-settings', route: { path: 'local-sip-settings', popup } }, {
        props: {
          component: {
            config: this.localConfig || {}
          },
          widget: new WidgetDefinition('localSipSettings', 'local SIP client settings', '')
            .paramGroup(pg('sipCredentials', 'SIP Credentials'), [
              pt('username', 'Username', 'SIP Username'),
              pt('password', 'Password', 'SIP Password'),
              pt('ownSipAddress', 'Own SIP Address', 'SIP Address (phone number) of this client. Used by the client to remove itself from the dial popup.')
            ])
        }
      })
      this.$f7.once('widgetConfigUpdate', this.storeLocalConfig)
    },
    storeLocalConfig (config) {
      this.localConfig = config
      localStorage.setItem('openhab.ui:sipConfig', JSON.stringify(this.localConfig))
      this.sipStart() // reload config
    },
    dial () {
      if (this.config.phonebook !== undefined && this.phonebook.size <= 1) {
        this.call(this.config.phonebook.split('=')[0])
      } else if (this.phonebook.size > 1) {
        const actionsPromise = new Promise((resolve, reject) => {
          if (this.phonebook.size > 0) {
            resolve(Array.from(this.phonebook.keys()).filter((key) => this.config.ownSipAddress ? this.config.ownSipAddress !== key : true).map((key) => {
              return {
                text: this.phonebook.get(key) || key,
                color: 'blue',
                onClick: () => {
                  this.call(key)
                }
              }
            }))
          }
        })
        actionsPromise.then((actions) => {
          this.$f7.actions.create({
            buttons: [
              actions,
              [{ text: 'Cancel', color: 'red' }]
            ]
          }).open()
        })
      } else {
        this.$f7.dialog.alert('Please configure phonebook entries')
      }
    }
  }
}
</script>
