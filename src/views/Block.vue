<template>
  <div style="top: 0;" id="blockdetails" class="page active">
    <block-state :static="true"></block-state>
  </div>
</template>

<script>
import BlockState from '@/components/BlockState.vue'

export default {
  name: 'Block',
  components: {
    BlockState
  },
  data() {
    return {
      blockdetails: null
    }
  },
  watch: {
  },
  mounted() {
    this.blockdetails = this.$route.params.hash
  }
}
</script>

<style lang="scss">
</style>