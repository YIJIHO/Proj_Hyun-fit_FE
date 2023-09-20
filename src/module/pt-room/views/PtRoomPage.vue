<template>
  <div id="ptRoom">
    <div class="flex justify-between items-center mt-12 mb-5 ml-16 mr-16">
      <div class="currentDate">
        <CurrentDate></CurrentDate>
      </div>
      <div class="w-76 h-11">
        <img src="/src/assets/images/mainLogo.png" />
      </div>
      <div
        class="buttonContainer flex justify-between items-center"
        v-if="bottonAfterJoin"
      >
        <input
          class="button"
          type="button"
          @click="joinSession('17', '123')"
          value="상담생성(트레이너)"
        />
        <input
          class="button"
          type="button"
          @click="joinSession('0', '123')"
          value="상담시작(사용자)"
        />
      </div>
      <div class="ressoningTime">
        <RessoningTime></RessoningTime>
      </div>
    </div>
    <div id="ptCamContainer" class="flex flex-col justify-center">
      <div
        id="after-join"
        class="flex justify-evenly"
        v-if="showAfterJoin"
        ref="afterJoin"
      >
        <div id="publisher"></div>
        <div id="subscriber"></div>
      </div>
    </div>
    <div id="controllerButtonContainer" class="flex justify-start mt-6">
      <div class="currentTime flex items-center">
        <CurrentTime></CurrentTime>
      </div>
      <div class="flex justify-start align-middle">
        <v-col cols="auto">
          <v-btn
            icon="mdi-plus"
            size="large"
            color="red"
            @click="leaveSession()"
            ><img src="/src/assets/images/phone-hangup.png" alt=""
          /></v-btn>
        </v-col>
        <v-col cols="auto">
          <v-btn icon="mdi-plus" size="large" color="grey" @click="toggleVideo"
            ><img :src="videoImgPath" alt=""
          /></v-btn>
        </v-col>
        <v-col cols="auto">
          <v-btn icon="mdi-plus" size="large" color="grey" @click="toggleAudio"
            ><img :src="audioImgPath" alt=""
          /></v-btn>
        </v-col>
      </div>
    </div>
  </div>
</template>

<script setup>
import CurrentTime from '../components/CurrentTimeComponent.vue'
import CurrentDate from '../components/CurrentDateComponent.vue'
import RessoningTime from '../components/RessoningTimeComponent.vue'
</script>
<script>
import { OpenVidu } from 'openvidu-browser'
import axios from 'axios'

export default {
  data() {
    return {
      OV: null,
      session: null,
      subscriber: [],
      APPLICATION_SERVER_URL: 'http://localhost:8080/',
      showAfterJoin: true,
      bottonAfterJoin: true,
      isAudioMuted: false,
      videoImgPath: '/src/assets/images/Vector.png',
      audioImgPath: '/src/assets/images/microphone.png',
    }
  },
  methods: {
    async getToken(currUserRole, mySessionId) {
      if (currUserRole == 0) {
        return await this.createToken(mySessionId)
      } else {
        const sessionId = await this.createSession(mySessionId)
        return await this.createToken(sessionId)
      }
    },
    async createSession(ptSeq) {
      const response = await axios.post(
        'http://localhost:8080/openvidu/sessions',
        { ptSeq },
        {
          headers: { 'Content-Type': 'application/json' },
          'Access-Control-Allow-Origin': 'http://localhost:8080',
        }
      )
      return response.data.sessionId
    },

    async createToken(sessionId) {
      const response = await axios.post(
        'http://localhost:8080/openvidu/sessions/' + sessionId + '/connections',
        {},
        {
          headers: { 'Content-Type': 'application/json' },
          'Access-Control-Allow-Origin': 'http://localhost:8080',
        }
      )
      return response.data.token
    },

    joinSession(currUserRole, mySessionId) {
      this.OV = new OpenVidu()
      this.session = this.OV.initSession()
      this.session.on('streamCreated', ({ stream }) => {
        let subscriber = this.session.subscribe(stream, 'subscriber')
        this.subscriber.push(subscriber)
      })
      this.session.on('exception', ({ exception }) => {
        console.warn(exception)
      })
      this.bottonAfterJoin = false
      this.getToken(currUserRole, mySessionId).then(token => {
        this.session
          .connect(token)
          .then(() => {
            let publisher = this.OV.initPublisher('publisher', {
              audioSource: undefined,
              videoSource: undefined,
              publishAudio: true,
              publishVideo: true,
              resolution: '',
              frameRate: 30,
              insertMode: 'APPEND',
              mirror: false,
            })
            this.mainStreamManager = publisher
            this.publisher = publisher
            this.session.publish(publisher)
            const videoElement = document.createElement('video')
            videoElement.classList.add('custom-video')
          })
          .catch(error => {
            console.log(
              'There was an error connecting to the session:',
              error.code,
              error.message
            )
          })
      })
    },

    async toggleVideo() {
      this.videoImgPath =
        this.videoImgPath === '/src/assets/images/Vector.png'
          ? '/src/assets/images/video-off-svgrepo-com.png'
          : '/src/assets/images/Vector.png'
      this.videoImgPath === '/src/assets/images/Vector.png'
        ? this.publisher.publishVideo(true)
        : this.publisher.publishVideo(false)
    },
    async toggleAudio() {
      this.audioImgPath =
        this.audioImgPath === '/src/assets/images/microphone.png'
          ? '/src/assets/images/mute-1-svgrepo-com.png'
          : '/src/assets/images/microphone.png'

      this.audioImgPath === '/src/assets/images/microphone.png'
        ? this.publisher.publishAudio(true)
        : this.publisher.publishAudio(false)
    },

    leaveSession() {
      if (this.session) this.session.disconnect()

      this.session = undefined
      this.mainStreamManager = undefined
      this.publisher = undefined
      this.subscriber = undefined
      this.OV = undefined

      window.removeEventListener('beforeunload', this.leaveSession)
    },
  },
  beforeUnmount() {
    if (this.session) this.session.disconnect()
  },
}
</script>

<style>
#ptRoom {
  width: 100%;
  height: 100vh;
  background-color: black;
}
#ptCamContainer {
  height: 70%;
}
#after-join {
  display: flex;
  flex-direction: row;
}
#publisher,
#subscriber {
  width: 45%;
  border-radius: 5%;
}
#margin {
  width: 5%;
  height: 100%;
}
.currentTime {
  margin-left: 70px;
  margin-right: 559px;
}
video {
  width: 100%;
  border-radius: 5%;
}
#controllerButtonContainer {
  height: 15%;
  position: relative;
  text-align: center;
}
/*테스트 후 삭제*/
.button {
  width: 100px;
  height: 50px;
  border-radius: 10%;
  background-color: white;
}
</style>