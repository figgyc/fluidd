<template>
  <v-app class="fluidd">
    <app-bar></app-bar>

    <FlashMessage
      v-model="flashMessage.open"
      :text="flashMessage.text"
      :type="flashMessage.type"
      :timeout="flashMessage.timeout"
    />

    <v-main>
      <router-view v-if="socketConnected" />
      <socket-disconnected-widget v-if="!socketConnected"></socket-disconnected-widget>
    </v-main>

    <app-footer></app-footer>
  </v-app>
</template>

<script lang="ts">
import { Component, Mixins } from 'vue-property-decorator'
import EventBus from '@/eventBus'
import UtilsMixin from './mixins/utils'
import { FlashMessage as FlashMessageType } from '@/types'
import AppBar from '@/components/AppBar.vue'
import AppFooter from '@/components/AppFooter.vue'
import SocketDisconnectedWidget from '@/components/widgets/SocketDisconnectedWidget.vue'
import FlashMessage from '@/components/FlashMessage.vue'
// import { MetaInfo } from 'vue-meta'

@Component({
  components: {
    AppBar,
    SocketDisconnectedWidget,
    FlashMessage,
    AppFooter
  }
  // metaInfo (this: App): MetaInfo {
  //   const instanceName = this.instanceName
  //   const progress = this.progress
  //   const r = {
  //     title: '',
  //     titleTemplate: ''
  //   }
  //   if (this.printerPrinting) {
  //     r.titleTemplate = `[${progress}%] | %s | ${instanceName}`
  //   } else {
  //     r.titleTemplate = `%s | ${instanceName}`
  //   }
  //   return r
  // }
})
export default class App extends Mixins(UtilsMixin) {
  flashMessage: FlashMessageType = {
    open: false,
    text: undefined,
    type: undefined
  }

  mounted () {
    this.$vuetify.theme.dark = this.$store.state.config.fileConfig.general.darkMode
    EventBus.$on('flashMessage', (payload: FlashMessageType) => {
      this.flashMessage.text = (payload && payload.text) || undefined
      this.flashMessage.type = (payload && payload.type) || undefined
      this.flashMessage.timeout = (payload && payload.timeout !== undefined) ? payload.timeout : undefined
      this.flashMessage.open = true
    })
  }
}
</script>

<style lang="scss" scoped>
  .title {
    background: linear-gradient(45deg, #1970b5, #9accf5);
    background-clip: text;
    -webkit-text-fill-color: transparent;
  }

  .logo {
    margin-right: 12px;
    max-height: 40px;
    max-width: 40px;
    object-fit: contain;
  }
</style>
