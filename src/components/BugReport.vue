<template>
  <div class="bug-report" style="margin:0">
    <div class="vue-grid col-2 default-gap">
      <VueFormField
        :title="i18n('version-title')"
        :subtitle="i18n('version-subtitle')"
      >
        <VueTypeAhead
          v-model="attrs.version"
          :suggestions="suggestions"
          :loading="loadingVersion"
          show-all
          show-max="30"
          restrict-choice
          required
        />
      </VueFormField>

      <VueFormField
        v-if="repo !== 'vuejs/vue-devtools'"
        :title="i18n('repro-title')"
      >
        <VueInput
          type="url"
          v-model="attrs.reproduction"
          required
        />

        <i18n
          slot="subtitle"
          id="repro-subtitle"
          @click-modal="show = true"
        />
      </VueFormField>

      <VueFormField
        v-else
        :title="i18n('browser-and-os-title')"
      >
        <VueInput
          v-model="attrs.browserAndOS"
          required
        />

        <i18n
          slot="subtitle"
          id="browser-and-os-subtitle"
        />
      </VueFormField>

      <VueFormField
        class="span-2"
        :title="i18n('steps-title')"
      >
        <VueInput
          type="textarea"
          rows="4"
          v-model="attrs.steps"
          required
        />
        <i18n slot="subtitle" id="steps-subtitle"/>
      </VueFormField>

      <VueFormField
        :title="i18n('expected-title')"
      >
        <VueInput
          type="textarea"
          rows="4"
          v-model="attrs.expected"
          required
        />
      </VueFormField>

      <VueFormField
        :title="i18n('actual-title')"
      >
        <VueInput
          type="textarea"
          rows="4"
          v-model="attrs.actual"
          required
        />
      </VueFormField>

      <VueFormField
        class="span-2"
        :title="i18n('extra-title')"
        :subtitle="i18n('extra-subtitle')"
      >
        <VueInput
          type="textarea"
          rows="4"
          v-model="attrs.extra"
        />
      </VueFormField>
    </div>

    <VueModal
      v-if="show"
      :title="i18n('repro-modal-title')"
      class="medium"
      @close="show = false"
    >
      <div class="default-body">
        <i18n id="repro-modal"/>
      </div>
    </VueModal>
  </div>
</template>

<script>
import { versionCompare, generate, updateQuery } from '../helpers'

export default {
  props: ['repo'],

  data () {
    return {
      show: false,
      attrs: {
        version: '',
        reproduction: '',
        steps: '',
        expected: '',
        actual: '',
        extra: '',
        browserAndOS: ''
      },
      versions: [],
      loadingVersion: false,
    }
  },

  computed: {
    suggestions () {
      return this.versions.slice()
        .sort((a, b) => -versionCompare(a.value, b.value))
    },
  },

  watch: {
    repo (repo) {
      this.versions = []
      this.attrs.version = ''
      this.fetchVersions()
    }
  },

  created () {
    this.fetchVersions()
  },

  methods: {
    async fetchVersions (page = 1) {
      this.loadingVersion = true
      const repo = this.repo
      const response = await fetch(`https://api.github.com/repos/${repo}/releases?page=${page}&per_page=100`)
      const releases = await response.json()

      if (this.repo !== repo) return

      if (!releases || !(releases instanceof Array)) return false

      this.versions = this.versions.concat(releases.map(
        r => ({ value: /^v/.test(r.tag_name) ? r.tag_name.substr(1) : r.tag_name })
      ))

      const link = response.headers.get('Link')

      if (link && link.indexOf('rel="next"') > -1) {
        await this.fetchVersions(page + 1)
      } else {
        this.loadingVersion = false
      }
    },

    generate () {
      const { version, reproduction, steps, expected, actual, extra, browserAndOS } = this.attrs

      return generate(`
### Version
${version}

${
  reproduction ? `### Reproduction link
[${reproduction}](${reproduction})` : ``}${
  browserAndOS ? `### Browser and OS info
${browserAndOS}` : ``
}

### Steps to reproduce
${steps}

### What is expected?
${expected}

### What is actually happening?
${actual}

${extra ? `---\n${extra}` : ''}
  `.trim())
    }
  },
}
</script>
