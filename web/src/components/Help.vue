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
      const base_url = process.env.VUE_APP_BASE_URL || `${location.origin}`;
      const path_prefix = process.env.VUE_APP_PATH_PREFIX || '';
      return help.replace(
        /http:\/\/localhost:8000/g,
        base_url + path_prefix
      );
    }
  }
}
</script>
