<template>
  <ul>
    <f7-list-item :title="title || 'Thing'" smart-select :smart-select-params="smartSelectParams" ref="smartSelect" v-if="ready">
      <select :name="name" :multiple="multiple" @change="select" :required="required">
        <option v-if="!multiple" value="" />
        <option v-for="thing in things" :value="thing.UID" :key="thing.UID" :selected="(multiple) ? value.indexOf(thing.UID) >= 0 : value === thing.UID">
          {{ thing.label }}
        </option>
      </select>
    </f7-list-item>
    <!-- for placeholder purposes before items are loaded -->
    <f7-list-item link v-show="!ready" :title="title" />
  </ul>
</template>

<script>
export default {
  props: ['title', 'name', 'value', 'multiple', 'required', 'filterType', 'filterUid', 'openOnReady'],
  data () {
    return {
      ready: false,
      things: [],
      icons: {},
      smartSelectParams: {
        view: this.$f7.view.main,
        openIn: 'popup',
        searchbar: true,
        searchbarPlaceholder: this.$t('dialogs.search.things'),
        virtualList: true,
        virtualListHeight: (this.$theme.aurora) ? 32 : undefined
      }
    }
  },
  created () {
    this.smartSelectParams.closeOnSelect = !(this.multiple)
    // TODO use a Vuex store
    this.$oh.api.get('/rest/things').then((data) => {
      this.things = data.sort((a, b) => {
        const labelA = a.label
        const labelB = b.label
        return labelA.localeCompare(labelB)
      })
      if (this.filterType) {
        this.things = this.things.filter((i) => this.filterType.indexOf(i.thingTypeUID) >= 0)
        if (this.things.length < 5) {
          this.smartSelectParams.openIn = 'sheet'
          this.smartSelectParams.searchbar = false
        }
      }
      if (this.filterUid && this.filterUid.length) {
        this.things = this.things.filter((t) => this.filterUid.indexOf(t.UID) >= 0)
      }
      this.ready = true
      if (this.openOnReady) {
        this.$nextTick(() => {
          this.$refs.smartSelect.f7SmartSelect.open()
        })
      }
    })
  },
  methods: {
    open () {
      this.$refs.smartSelect.f7SmartSelect.open()
    },
    select (e) {
      this.$f7.input.validateInputs(this.$refs.smartSelect.$el)
      this.$emit('input', e.target.value)
      this.$f7.emit('thingPicked', e.target.value)
    }
  }
}
</script>
