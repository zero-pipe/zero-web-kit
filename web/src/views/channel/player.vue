<template>
  <div id="devicePlayer">

    <el-dialog
      v-if="showVideoDialog"
      v-el-drag-dialog
      title="视频播放"
      top="10vh"
      width="65vw"
      :close-on-click-modal="false"
      :visible.sync="showVideoDialog"
      @close="close()"
    >
      <div class="dhsdk-player-body">

        <div class="player-side">
          <div
            class="player-container"
            :style="{ height: playerHeight }"
            v-loading="isLoging"
            element-loading-text="正在邀请设备推流…"
            element-loading-background="rgba(0, 0, 0, 0.72)"
          >
            <div v-if="playError" class="player-error-tip">{{ playError }}</div>
            <playerTabs ref="playerTabs" :has-audio="hasAudio" :show-button="true"
              @playerChanged="playerChanged" />
          </div>
        </div>

        <div class="control-side">
          <channelPtzPanel
            :channel-id="channelId"
            @drag-zoom-start="toggleDragZoom"
          />

          <el-tabs v-model="tabActiveName" @tab-click="tabHandleClick" class="control-tabs">
            <el-tab-pane label="预置位" name="preset">
              <channelPreset
                v-if="tabActiveName === 'preset'"
                :channel-id="channelId"
                style="margin-top: 8px;"
              />
            </el-tab-pane>
            <el-tab-pane label="实时视频" name="media">
              <streamMediaPanel v-if="tabActiveName === 'media'" :player-url="playerUrlInfo.playerUrl" :play-url="playerUrlInfo.playUrl" :stream-info="streamInfo" />
            </el-tab-pane>
            <el-tab-pane label="编码信息" name="codec">
              <mediaInfo v-if="tabActiveName === 'codec'" ref="mediaInfo" :app="app" :stream="streamId" :media-server-id="mediaServerId" />
            </el-tab-pane>
          </el-tabs>
        </div>

      </div>
    </el-dialog>
  </div>
</template>

<script>
import elDragDialog from '@/directive/el-drag-dialog'
import playerTabs from '../common/playerTabs.vue'
import channelPtzPanel from './common/channelPtzPanel.vue'
import channelPreset from './common/ptzPreset.vue'
import mediaInfo from '../common/mediaInfo.vue'
import streamMediaPanel from '../common/streamMediaPanel.vue'

export default {
  name: 'ChannelPlayer',
  directives: { elDragDialog },
  components: { playerTabs, channelPtzPanel, channelPreset, mediaInfo, streamMediaPanel },
  props: {},
  data() {
    return {
      videoUrl: '',
      streamId: '',
      app: '',
      mediaServerId: '',
      channelId: '',
      tabActiveName: 'preset',
      hasAudio: false,
      isLoging: false,
      playError: '',
      showVideoDialog: false,
      streamInfo: null,
      playerHeight: '48vh',
      playerUrlInfo: {
        playerUrl: null,
        playUrl: null
      },
      dragZoomDirection: ''
    }
  },
  methods: {
    tabHandleClick(tab) {
      if (tab.name === 'codec') {
        this.$refs.mediaInfo && this.$refs.mediaInfo.startTask()
      } else {
        this.$refs.mediaInfo && this.$refs.mediaInfo.stopTask()
      }
    },
    openDialog(tab, channelId, param) {
      if (this.showVideoDialog) return
      this.tabActiveName = tab || 'preset'
      this.channelId = channelId
      this.streamId = ''
      this.mediaServerId = ''
      this.app = ''
      this.videoUrl = ''
      this.playError = ''
      this.showVideoDialog = true
      if (param && param.pending) {
        this.streamInfo = null
        this.hasAudio = !!(param.hasAudio)
        this.isLoging = true
        return
      }
      if (param && param.streamInfo) {
        this.play(param.streamInfo, param.hasAudio)
      }
    },
    onStreamReady(streamInfo, hasAudio) {
      if (!this.showVideoDialog) return
      this.play(streamInfo, hasAudio)
    },
    onStreamError(error) {
      if (!this.showVideoDialog) return
      this.isLoging = false
      const msg = typeof error === 'string' ? error : (error && error.message) || '取流失败，请稍后重试'
      this.playError = msg
      this.$message({ showClose: true, message: msg, type: 'error' })
    },
    play(streamInfo, hasAudio) {
      this.streamInfo = streamInfo
      this.hasAudio = hasAudio
      this.isLoging = false
      this.playError = ''
      this.streamId = streamInfo.stream
      this.app = streamInfo.app
      this.mediaServerId = streamInfo.mediaServerId
      this.showVideoDialog = true
      this.$nextTick(() => {
        if (this.$refs.playerTabs) {
          this.$refs.playerTabs.setStreamInfo(streamInfo.transcodeStream || streamInfo)
        }
      })
    },
    playerChanged(playerUrlInfo) {
      this.playerUrlInfo = playerUrlInfo
    },
    close() {
      if (this.$refs.playerTabs) {
        this.$refs.playerTabs.stop()
      }
      this.isLoging = false
      this.playError = ''
      this.streamInfo = null
      this.videoUrl = ''
      this.showVideoDialog = false
    },
    toggleDragZoom(direction) {
      this.dragZoomDirection = direction
      this.$refs.playerTabs.startDragZoom((params) => {
        params.channelId = this.channelId
        const action = this.dragZoomDirection === 'in' ? 'commonChanel/dragZoomIn' : 'commonChanel/dragZoomOut'
        const successMsg = this.dragZoomDirection === 'in' ? '拉框放大成功' : '拉框缩小成功'
        const failMsg = this.dragZoomDirection === 'in' ? '拉框放大失败' : '拉框缩小失败'
        this.$store.dispatch(action, params).then(() => {
          this.$message({ showClose: true, message: successMsg, type: 'success' })
        }).catch(() => {
          this.$message({ showClose: true, message: failMsg, type: 'error' })
        })
        this.dragZoomDirection = ''
      })
    }
  }
}
</script>

<style>
#devicePlayer .el-dialog__body { padding: 10px 20px; }
.dhsdk-player-body { display: flex; gap: 16px; }
.player-side { flex: 3; min-width: 0; }
.player-container { width: 100%; position: relative; }
.player-error-tip {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 2;
  color: #f56c6c;
  font-size: 14px;
  text-align: center;
  padding: 0 12px;
}
.control-side { flex: 2; min-width: 340px; display: flex; flex-direction: column; }
.control-tabs { flex: 1; display: flex; flex-direction: column; min-height: 220px }
.control-tabs .el-tabs__content { flex: 1; overflow: auto; }
.media-info-content { overflow: auto; }
.media-row { display: flex; margin-bottom: 0.5rem; height: 2.5rem; }
.media-label { width: 6rem; line-height: 2.5rem; text-align: right; flex-shrink: 0; }
</style>
