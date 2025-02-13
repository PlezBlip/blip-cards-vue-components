<template>
  <bds-grid direction="row" justifyContent="space-between" gap="2" class="container-reply-item">
    <bds-grid direction="row" gap="1" align-items="center">
      <bds-grid direction="column" padding="1" gap="1">
        <bds-typo variant="fs-14" bold="bold" :margin="false" v-if="replyingToText" class="typo text-replying">{{replyingToText}}</bds-typo>
        <bds-grid direction="row" gap="1" >
          <bds-grid>
            <bds-icon theme="outline" name="video" size="medium" class="typo" />
          </bds-grid>
          <bds-grid direction="column" >
            <bds-typo tag="span" variant="fs-14" bold="regular" class="typo">{{ timeVideo || translations.video }}</bds-typo>
            <bds-typo tag="span" variant="fs-14" bold="regular" class="typo multiline-text-overflow-ellipsis"
              v-if="replyValueHasText">{{ this.inReplyTo.value.text }}</bds-typo>
          </bds-grid>
        </bds-grid>
      </bds-grid>
    </bds-grid>
    
    <bds-grid>
      <video
        :src="videoUri"
        ref="video"
        @loadedmetadata="captureThumbnail"
        class="video-replied"
        :class="replyValueHasText ? 'full-height' : 'simplified'"
      />
    </bds-grid>
  </bds-grid>
</template>

<script>
import { tryCreateLocalMediaUri, isAuthenticatedMediaLink } from '../../../utils/media.js'

export default {
  name: 'in-reply-to-video',
  mixins: [],
  props: {
    inReplyTo: {
      type: Object,
      default: () => ({})
    },
    replyingToText: {
      type: String,
      default: null
    },
    isReplyingOnInput: {
      type: Boolean,
      default: true
    },
    translations: {
      type: Object,
      default: () => ({})
    },
    asyncFetchMedia: {
      type: Function
    }
  },
  data() {
    return {
      videoUri: undefined,
      thumbnail: undefined,
      timeVideo: undefined
    }
  },
  computed: {
    replyValueHasText() {
      return this.inReplyTo.value && this.inReplyTo.value.text
    }
  },
  mounted: async function() {
    const document = this.inReplyTo.value
    const uri = isAuthenticatedMediaLink(document)
      ? await tryCreateLocalMediaUri(this.inReplyTo.value, this.asyncFetchMedia)
      : document.uri

    this.initVideo(uri)
  },
  methods: {
    initVideo(uri) {
      this.videoUri = uri
    },
    captureThumbnail() {
      const video = this.$refs.video

      video.currentTime = 2
      this.timeVideo = this.formatDuration(video.duration)

      video.addEventListener('seeked', () => {
        this.canvasRendered = true
        this.$nextTick(() => {
          const canvas = document.createElement('canvas')
          const context = canvas.getContext('2d')
          context.drawImage(video, 0, 0, 80, 80)
        })
      })
    },
    formatDuration(seconds) {
      const minutes = Math.floor(seconds / 60)
      const remainingSeconds = Math.floor(seconds % 60)

      const formattedSeconds = remainingSeconds < 10 ? `0${remainingSeconds}` : remainingSeconds

      return `${minutes}:${formattedSeconds}`
    }
  }
}
</script>

<style scoped>
  .video-replied {  
    margin-right: -4px;
    border-radius: 0 !important;
  }

  .simplified {
    max-height: 95px;
  }

  .full-height {
    max-height: 110px;
  }
</style>