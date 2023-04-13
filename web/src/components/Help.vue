<template>
  <vue-markdown>
    {{ help() }}
  </vue-markdown>
</template>

<script>
import VueMarkdown from 'vue-markdown'

import help from '@/assets/getting-started.md'

export default {
  name: 'help',
  components: {
    VueMarkdown
  },
  methods: {
    help() {
      const port = process.env.VUE_APP_BACKEND_PORT || location.port
      const host = process.env.VUE_APP_BACKEND_HOST || location.hostname
      const path_prefix = process.env.VUE_APP_BACKEND_PATH_PREFIX || ''

      return help.replace(
        /http:\/\/localhost:8000/g,
        `${location.protocol}//${host}${port !== '' ? ':' + port : ''}${path_prefix}`
      );
    }
  }
}
</script>
