<template lang="pug">
  input(type="checkbox" :value="val" v-model="checked" @change="onChange")
</template>

<script>
export default {
  props: [ 'value', 'val' ],
  data () {
    return {
      checkedProxy: false
    }
  },
  computed: {
    checked: {
      get () {
        return this.value
      },
      set (val) {
        this.checkedProxy = val
      }
    }
  },
  methods: {
    onChange (e) {
      this.$emit('input', this.checkedProxy)
    }
  }
}
</script>
