<template>
  <div id="vuepress-theme-blog__global-layout">
    <Navbar
      @toggle-sidebar="isMobileHeaderOpen = !isMobileHeaderOpen"
    />
    <Sidebar
      :is-open="isMobileHeaderOpen"
    />
    <div class="content-wrapper" @click="isMobileHeaderOpen = false">
      <DefaultGlobalLayout />
    </div>
    <Footer />
  </div>
</template>

<script>
import GlobalLayout from '@app/components/GlobalLayout.vue'
import Navbar from '@theme/components/Navbar.vue'
import Sidebar from '@theme/components/Sidebar.vue'
import Footer from '@theme/components/Footer.vue'
export default {
  components: {
    DefaultGlobalLayout: GlobalLayout,
    Navbar,
    Sidebar,
    Footer,
  },
  data() {
    return {
      isMobileHeaderOpen: false,
    }
  },
  mounted() {
    this.$router.afterEach(() => {
      this.isMobileHeaderOpen = false
    })
  },
}
</script>

<style lang="stylus">
#vuepress-theme-blog__global-layout
  word-wrap break-word
.content-wrapper
  padding 160px 15px 80px 15px
  min-height calc(100vh - 80px - 60px - 160px)
  max-width $contentWidth
  margin 0 auto
  @media (max-width: $MQMobile)
    &
      padding 100px 15px 20px 15px
      min-height calc(100vh - 20px - 60px - 100px)
</style>