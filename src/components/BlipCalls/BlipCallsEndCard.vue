<template>
  <div>
    <div
      v-if="
        previewDocument.content != null && previewDocument.content.length > 0
      "
      :class="`calls-card ${isFailedMessage(status, position)}`.trim()"
    >
      <div :class="`bubble ${position}`">
        <div class="content">
          <div class="content__details">
            <div :class="`content__details__icon ${documentStatusClass}`">
              <bds-icon
                :name="iconName"
                color="var(--color-content-default, #454545)"
              />
            </div>
            <div class="content__details__text">
              <div class="content__details__text__title">
                <bds-typo
                  class="typo title"
                  variant="fs-16"
                  bold="bold"
                  margin="false"
                  >{{ titleText }}
                </bds-typo>
                <div v-if="canDownloadRecording">
                  <bds-dropdown position="bottom-right">
                    <div slot="dropdown-activator">
                      <bds-icon 
                        class="btn-download"
                        :color="position === 'right' ? 'var(--color-surface-0, #FFFFFF)' : 'var(--color-content-default, #454545)'"
                        name="arrow-down"
                      />
                    </div>
                    <div slot="dropdown-content">
                      <bds-list type-list="default">
                        <bds-list-item
                          :text="downloadRecordingLabel"
                          clickable
                          @click="onDownload"
                        ></bds-list-item>
                        <bds-list-item
                          v-if="transcription && transcription.callRecordingEnabled"
                          :text="transcribeRecordingLabel"
                          clickable
                          @click="onTranscribe"
                        ></bds-list-item>
                      </bds-list>
                    </div>
                  </bds-dropdown>
                </div>
                <div v-else v-html="identificationTextComponent" />
              </div>
              <div class="content__details__text__subtitle">
                <bds-typo
                  class="typo"
                  variant="fs-14"
                  bold="semi-bold"
                  margin="false"
                  >{{ statusText }}
                </bds-typo>
                <div v-if="canDownloadRecording" v-html="identificationTextComponent" />
              </div>
            </div>
          </div>
          <div
            v-if="isSuccess"
            :class="
              `content__record ${documentTypeClass} ${
                hasMediaUri ? 'has-media' : ''
              }`
            "
          >
            <template v-if="hasMediaUri && isVideoType">
              <blip-video
                video-uri-msg="videoUriMsg"
                :document="document.media"
                :full-document="fullDocument.media"
                :position="position"
                :date="date"
                @updated="emitUpdate"
                :editable="editable"
                :on-save="save"
                :on-deleted="onDeleted"
                :on-metadata-edit="isMetadataReady"
                :deletable="deletable"
                :on-cancel="onCancel"
                :editing="editing"
                :on-video-validate-uri="onMediaValidateUri"
                :async-fetch-media="asyncFetchMedia"
              />
            </template>
            <template v-else-if="hasMediaUri && isVoiceType">
              <blip-audio
                file-url-msg="fileUrlMsg"
                :document="document.media"
                :full-document="fullDocument.media"
                :position="position"
                :date="date"
                :editable="editable"
                :on-save="save"
                :on-deleted="onDeleted"
                :on-metadata-edit="isMetadataReady"
                :deletable="deletable"
                :on-cancel="onCancel"
                :editing="editing"
                :on-audio-validate-uri="onMediaValidateUri"
                :async-fetch-media="asyncFetchMedia"
              />
            </template>
            <div
              v-else-if="!hasMediaUri"
              class="loading-media-content"
              :class="refreshingMediaUri ? 'refreshing-media' : ''"
            >
              <bds-loading-spinner color="light" size="small" />
              <bds-typo variant="fs-14" bold="semi-bold" line-height="plus"
                >{{
                  refreshingMediaUri ? preparingRecordingMsg : loadRecordingMsg
                }}
              </bds-typo>
              <button
                class="btn-refresh"
                :disabled="refreshingMediaUri"
                @click="refreshMediaUrl"
              >
                <bds-icon
                  name="refresh"
                  size="x-large"
                  color="var(--color-surface-1, #F6F6F6)"
                />
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
    <blip-card-date
      :status="status"
      :position="position"
      :date="date"
      :failed-to-send-msg="failedToSendMsg"
    />
  </div>
</template>

<script>
import { default as base } from '../../mixins/baseComponent.js'
import { linkify, isFailedMessage } from '../../utils/misc.js'
import BlipAudio from '.././MediaLink/Audio'
import BlipVideo from '.././MediaLink/Video'
import blipCallsStatus from '../../enums/blipCallsStatus.enum.js'
import blipCallsType from '../../enums/blipCallsType.enum.js'

export default {
  name: 'blip-calls-end-card',
  mixins: [base],
  components: {
    BlipAudio,
    BlipVideo
  },
  props: {
    status: {
      type: String,
      default: ''
    },
    length: {
      type: Number,
      default: 532
    },
    disableLink: {
      type: Boolean,
      default: false
    },
    videoCallMsg: {
      type: String,
      default: 'Chamada de vídeo'
    },
    voiceCallMsg: {
      type: String,
      default: 'Ligação'
    },
    successStatusMsg: {
      type: String,
      default: 'Finalizada'
    },
    failedStatusMsg: {
      type: String,
      default: 'Não atendida'
    },
    cancelStatusMsg: {
      type: String,
      default: 'Cancelada'
    },
    notAnsweredStatusMsg: {
      type: String,
      default: 'Não atendida'
    },
    preparingRecordingMsg: {
      type: String,
      default: 'Preparando Gravação'
    },
    loadRecordingMsg: {
      type: String,
      default: 'Carregar Gravação'
    },
    failedToSendMsg: {
      type: String,
      default: 'Falha ao enviar a mensagem.'
    },
    downloadRecordingLabel: {
      type: String,
      default: 'Baixar gravação'
    },
    transcribeRecordingLabel: {
      type: String,
      default: 'Ver transcrição'
    },
    onMediaValidateUri: {
      type: Function
    },
    asyncFetchMedia: {
      type: Function
    },
    onAsyncFetchSession: {
      type: Function
    },
    transcription: {
      type: Object
    }
  },
  data() {
    return {
      refreshingMediaUri: false,
      hasMediaUri: false,
      isFailedMessage
    }
  },
  mounted() {
    this.refreshMediaUrl()
  },
  computed: {
    previewDocument: function() {
      const sanitizedDocument = this.sanitize(this.document)

      return {
        hasPreview: sanitizedDocument.length > this.length,
        previewContent: linkify(
          sanitizedDocument.substring(0, this.length - 3) + '...',
          this.disableLink
        ),
        content: linkify(sanitizedDocument, this.disableLink)
      }
    },
    iconName: function() {
      const failedVideoIconName = 'video-ended'
      const failedVoiceIconName = 'voip-ended'

      const icons = {
        [blipCallsType.video]: {
          [blipCallsStatus.answer]: 'video-calling',
          [blipCallsStatus.completed]: 'video-calling',
          [blipCallsStatus.busy]: failedVideoIconName,
          [blipCallsStatus.cancel]: failedVideoIconName,
          [blipCallsStatus.noAnswer]: failedVideoIconName,
          [blipCallsStatus.progress]: failedVideoIconName,
          [blipCallsStatus.failed]: failedVideoIconName,
          [blipCallsStatus.unknown]: failedVideoIconName
        },
        [blipCallsType.voice]: {
          [blipCallsStatus.answer]: this.isOutboundCall
            ? 'voip-calling'
            : 'voip-receiving',
          [blipCallsStatus.completed]: this.isOutboundCall
            ? 'voip-calling'
            : 'voip-receiving',
          [blipCallsStatus.busy]: failedVoiceIconName,
          [blipCallsStatus.cancel]: failedVoiceIconName,
          [blipCallsStatus.noAnswer]: failedVoiceIconName,
          [blipCallsStatus.progress]: failedVoiceIconName,
          [blipCallsStatus.failed]: failedVoiceIconName,
          [blipCallsStatus.unknown]: failedVoiceIconName
        }
      }

      return icons[this.document.type][this.document.status]
    },
    titleText: function() {
      const msgs = {
        [blipCallsType.video]: this.videoCallMsg,
        [blipCallsType.voice]: this.voiceCallMsg
      }

      return this.sanitize(msgs[this.document.type])
    },
    identificationText: function() {
      return this.sanitize(
        this.maskIdentification(this.document.identification)
      )
    },
    statusText: function() {
      const statusMessage = {
        [blipCallsStatus.answer]: this.successStatusMsg,
        [blipCallsStatus.busy]: this.notAnsweredStatusMsg,
        [blipCallsStatus.cancel]: this.cancelStatusMsg,
        [blipCallsStatus.noAnswer]: this.notAnsweredStatusMsg,
        [blipCallsStatus.progress]: this.failedStatusMsg,
        [blipCallsStatus.unknown]: this.notAnsweredStatusMsg,
        [blipCallsStatus.failed]: this.cancelStatusMsg,
        [blipCallsStatus.completed]: this.successStatusMsg
      }

      return this.sanitize(statusMessage[this.document.status])
    },
    isSuccess: function() {
      return [blipCallsStatus.answer, blipCallsStatus.completed].some(
        (status) => status === this.document.status
      )
    },
    isVideoType: function() {
      return this.document.type === blipCallsType.video
    },
    isVoiceType: function() {
      return this.document.type === blipCallsType.voice
    },
    documentTypeClass: function() {
      return this.document.type.toLowerCase()
    },
    documentStatusClass: function() {
      return this.document.status.toLowerCase()
    },
    isOutboundCall: function() {
      return this.document.direction
        ? this.document.direction.toLowerCase() === 'outbound'
        : true
    },
    canDownloadRecording: function() {
      return this.isSuccess && this.hasMediaUri && this.isVoiceType
    },
    identificationTextComponent: function() {
      return `<bds-typo
        v-if="${this.document.identification}"
        class="typo"
        variant="fs-12"
        bold="regular"
        margin="false">${this.identificationText}</bds-typo>`
    }
  },
  methods: {
    async refreshMediaUrl() {
      try {
        if (this.document.media && this.document.media.uri) {
          this.hasMediaUri = true
          return
        }

        if (this.isSuccess && this.onAsyncFetchSession) {
          this.refreshingMediaUri = true

          const session = await this.onAsyncFetchSession(
            this.document.sessionId
          )

          if (session && session.recordedFileUrl) {
            this.document.media.uri = session.recordedFileUrl
            this.hasMediaUri = true
          } else {
            await new Promise((resolve) => {
              setTimeout(async () => {
                resolve()
              }, 5000)
            })
          }
        }
      } finally {
        this.refreshingMediaUri = false
      }
    },
    maskIdentification(identification) {
      if (!identification) return ''

      if (this.isVideoType) {
        const sequentialId = identification.replace(/\D/g, '')

        return `#${sequentialId}`
      } else {
        const mask = this.getPhoneMask(identification)

        const digits = identification.replace(/\D/g, '')
        let i = 0

        return mask
          ? mask.replace(/#/g, (_) => (digits[i] ? digits[i++] : ''))
          : identification
      }
    },
    getPhoneMask(phone, countryCode = 55) {
      if (!phone) return ''

      const digits = phone.replace(/\D/g, '')
      let mask

      switch (countryCode) {
        case 55:
          switch (digits.length) {
            case 10:
              mask = '(##) ####-####'
              break
            case 11:
              mask = '(##) #####-####'
              break
            case 12:
              mask = '+## (##) ####-####'
              break
            case 13:
              mask = '+## (##) #####-####'
              break
            default:
              mask = undefined
              break
          }
          break

        default:
          mask = undefined
          break
      }

      return mask
    },
    emitUpdate() {
      this.$emit('updated')
    },
    async onDownload() {
      const a = document.createElement('a')

      const response = await fetch(this.document.media.uri)
      const blob = await response.blob()

      const extension = this.isVoiceType ? 'webm' : 'mp4'
      const fileName = `${this.document.sessionId}.${extension}`

      const url = window.URL.createObjectURL(blob)

      a.style.display = 'none'
      a.target = '_blank'
      a.href = url
      a.download = fileName

      document.body.appendChild(a)

      a.click()

      document.body.removeChild(a)
      window.URL.revokeObjectURL(url)
    },
    async onTranscribe() {
      if (this.transcription && this.transcription.onOpenMfeModal) {
        await this.transcription.onOpenMfeModal({ url: this.document.media.uri, source: 'calls' })
      }
    }
  }
}
</script>

<style lang="scss">
@import '../../styles/variables.scss';

$space-0: var(--space-0, 0);
$space-05: var(--space-05, 0.25rem);
$space-1: var(--space-1, 0.5rem);
$space-2: var(--space-2, 1rem);
$space-4: var(--space-4, 2rem);
$space-5: var(--space-5, 2.5rem);
$space-6: var(--space-6, 3rem);
$default-transition: var(--default-transition, all 0.25s ease-in);

.blip-message-group {
  .blip-card-group {
    .calls-card {
      div.notification {
        display: flex !important;
      }
    }
  }
}

.calls-card {
  .bubble {
    padding: 12px;
    word-wrap: break-word;
    text-align: left;

    .btn-download {
      cursor: pointer;
      display: flex;
    }

    .content {
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: flex-center;
      gap: $space-1;
      width: auto;

      &__details {
        display: flex;
        flex-direction: row;
        justify-content: space-between;
        align-items: center;
        gap: 10px;
        padding: $space-1;

        min-height: 60px;

        &__icon {
          display: flex;
          flex-direction: row;
          justify-content: center;
          align-items: center;
          padding: $space-1;
          border-radius: $space-1;
          background-color: var(--color-error, #f99f9f);

          &.answer,
          &.completed {
            background-color: var(--color-success, #84ebbc);
          }
        }

        &__text {
          display: flex;
          flex-direction: column;
          justify-content: center;
          align-items: stretch;
          flex: 1;
          
          &__title {
            display: flex;
            flex-direction: row;
            justify-content: space-between;
            align-items: center;
            gap: 32px;
            flex: 1;
          }

          &__subtitle {
            display: flex;
            flex-direction: row;
            justify-content: space-between;
            align-items: center;
            gap: 16px;
            flex: 1;
          }
        }
      }

      &__record {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: start;
        gap: $space-05;
        align-self: stretch;
        padding: $space-05 12px;
        background-color: var(--color-content-disabled, #636363);
        border-radius: 6px 6px $space-0 6px;
        overflow: hidden;

        min-height: 60px;

        &.video {
          &.has-media {
            padding: 0;
            background-color: transparent;
          }

          > div {
            display: flex;
            flex: 1;
            align-self: stretch;

            > div {
              display: flex;
              flex: 1;
              align-self: stretch;

              div.video-player-wrapper {
                flex: 1;

                #blipVideo {
                  border-radius: $space-1 !important;
                }

                .video-player-controls {
                  margin-top: $space-2;
                  padding: $space-05 $space-2;
                }
              }
            }
          }
        }

        &.voice {
          > div {
            display: flex;
            align-self: stretch;

            div.audio-player-wrapper {
              flex: 1;

              .audio-play-pause {
                margin-top: 3px;
              }
            }
          }
        }

        .loading-media-content {
          display: flex;
          flex-direction: row;
          justify-content: space-between;
          align-items: center;
          align-self: stretch;
          gap: $space-1;

          bds-loading-spinner {
            display: flex;
            transition: $default-transition;
            overflow: hidden;

            opacity: 0;
            transform: translateX(-200%);
          }

          bds-typo {
            flex: 1;
            color: $color-surface-1;
            transition: $default-transition;
            transform: translateX(-40px);
          }

          button {
            &.btn-refresh {
              position: relative;
              display: flex;
              justify-content: center;
              align-items: center;
              margin: 0;
              padding: $space-05 $space-1;
              background-color: transparent;
              border-radius: $space-1;
              border: 1px solid $color-border-1;
              cursor: pointer;
              transition: $default-transition;
              opacity: 1;

              &::before {
                content: '';
                position: absolute;
                top: 0;
                left: 0;
                width: 100%;
                height: 100%;
                background-color: transparent;
                z-index: 0;
                border-radius: $space-1;
              }

              &:hover {
                &::before {
                  background-color: $color-hover;
                }
              }

              &:active {
                &::before {
                  background-color: $color-pressed;
                }
              }

              &:disabled {
                cursor: default;
                border: 1px solid $color-content-ghost;

                &::before {
                  background-color: transparent;
                }
              }
            }
          }

          &.refreshing-media {
            bds-loading-spinner {
              opacity: 1;
              transform: translateX(0);
            }

            bds-typo {
              transform: translateX(0);
            }

            button {
              &.btn-refresh {
                transform: translateX(200%);
                opacity: 0;
              }
            }
          }
        }
      }
    }

    &.left,
    &.middle {
      .content {
        &__record {
          border-radius: 6px 6px 6px $space-0;

          .audio-player-wrapper {
            .slider {
              background-color: $color-content-ghost;
            }

            .progress .pin {
              background-color: $color-surface-1;
            }

            .audio-player-button {
              fill: $color-surface-1;
            }

            .audio-player-time {
              color: $color-surface-1;
            }

            .blip-change-playback-rate {
              border-color: $color-content-ghost;
              color: $color-surface-1;
            }
          }
        }
      }
    }

    &.middle {
      .content {
        &__record {
          border-radius: 6px;
        }
      }
    }
  }
}

.blip-plain-text-metadata {
  margin-top: -20px;
  padding: 0 10px 10px 0;
}
</style>
