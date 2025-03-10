<!--
Copyright 2022 Google Inc. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<template>
  <div>
    <div class="pa-4" flat :class="$vuetify.theme.dark ? 'dark-hover' : 'light-hover'">
      <span style="cursor: pointer" @click="expanded = !expanded"
        ><v-icon left>mdi-tag-multiple-outline</v-icon> Tags (<small
          ><strong>{{ tags.length }}</strong></small
        >)</span
      >
    </div>

    <v-expand-transition>
      <div v-show="expanded && tags.length">
        <v-row
          no-gutters
          v-for="label in labels"
          :key="label.label"
          class="pa-2 pl-5"
          :class="$vuetify.theme.dark ? 'dark-hover' : 'light-hover'"
        >
          <div @click="searchForLabel(label.label)" style="cursor: pointer; font-size: 0.9em">
            <v-icon v-if="label.label === '__ts_star'" left small color="amber">mdi-star</v-icon>
            <v-icon v-if="label.label === '__ts_comment'" left small>mdi-comment-multiple-outline</v-icon>

            <span
              >{{ label.label | formatLabelText }} (<small
                ><strong>{{ label.count | compactNumber }}</strong></small
              >)</span
            >
          </div>
        </v-row>

        <v-row
          no-gutters
          v-for="tag in tags"
          :key="tag.tag"
          class="pa-2 pl-5"
          :class="$vuetify.theme.dark ? 'dark-hover' : 'light-hover'"
        >
          <div @click="searchForTag(tag.tag)" style="cursor: pointer; font-size: 0.9em">
            <span
              >{{ tag.tag }} (<small
                ><strong>{{ tag.count | compactNumber }}</strong></small
              >)</span
            >
          </div>
        </v-row>
      </div>
    </v-expand-transition>
    <v-divider></v-divider>
  </div>
</template>

<script>
import EventBus from '../../main'

const defaultQueryFilter = () => {
  return {
    from: 0,
    terminate_after: 40,
    size: 40,
    indices: '_all',
    order: 'asc',
    chips: [],
  }
}

export default {
  props: [],
  data: function () {
    return {
      expanded: false,
    }
  },
  computed: {
    sketch() {
      return this.$store.state.sketch
    },
    meta() {
      return this.$store.state.meta
    },
    tags() {
      return this.$store.state.tags
    },
    labels() {
      return this.meta.filter_labels
    },
  },
  methods: {
    searchForTag(tag) {
      let eventData = {}
      eventData.doSearch = true
      eventData.queryString = 'tag:' + '"' + tag + '"'
      eventData.queryFilter = defaultQueryFilter()
      EventBus.$emit('setQueryAndFilter', eventData)
    },
    searchForLabel(label) {
      let eventData = {}
      eventData.doSearch = true
      eventData.queryString = '*'
      eventData.queryFilter = defaultQueryFilter()
      let chip = {
        field: '',
        value: label,
        type: 'label',
        operator: 'must',
        active: true,
      }
      eventData.queryFilter.chips.push(chip)
      EventBus.$emit('setQueryAndFilter', eventData)
    },
  },
}
</script>
