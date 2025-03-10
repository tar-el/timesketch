<!--
Copyright 2021 Google Inc. All rights reserved.

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
  <span>
    <v-dialog v-if="timelineStatus === 'processing'" v-model="dialogStatus" width="600">
      <template v-slot:activator="{ on, attrs }">
        <v-chip v-on="on" :style="getTimelineStyle(timeline)" class="mr-2 mb-3">
          <span class="timeline-name-ellipsis">{{ timeline.name }}</span>
          <span class="ml-1">
            <v-progress-circular small indeterminate color="grey" :size="20" :width="1"></v-progress-circular>
          </span>
        </v-chip>
      </template>
      <v-card>
        <v-app-bar flat dense>Importing events to timeline "{{ timeline.name }}"</v-app-bar>
        <div class="pa-5">
          <ul>
            <li><strong>Opensearch index: </strong>{{ timeline.searchindex.index_name }}</li>
            <li v-if="timelineStatus === 'processing' || timelineStatus === 'ready'">
              <strong>Number of events: </strong>
              {{ allIndexedEvents | compactNumber }}
            </li>
            <li><strong>Created by: </strong>{{ timeline.user.username }}</li>
            <li>
              <strong>Created at: </strong>{{ timeline.created_at | shortDateTime }}
              <small>({{ timeline.created_at | timeSince }})</small>
            </li>
          </ul>
          <br />

          <div class="mb-3">{{ datasourcesProcessing.length }} datasource(s) in progress..</div>
          <v-alert
            v-for="datasource in datasourcesProcessing"
            :key="datasource.id"
            colored-border
            border="left"
            elevation="1"
            :color="datasourceStatusColors(datasource)"
          >
            <ul>
              <li><strong>Original filename:</strong> {{ datasource.original_filename }}</li>
              <li><strong>File on disk:</strong> {{ datasource.file_on_disk }}</li>
              <li><strong>File size:</strong> {{ datasource.file_size | compactBytes }}</li>
              <li><strong>Provider:</strong> {{ datasource.provider }}</li>
              <li v-if="datasource.data_label"><strong>Data label:</strong> {{ datasource.data_label }}</li>
              <li><strong>Status:</strong> {{ dataSourceStatus(datasource) }}</li>
              <li>
                <strong>Total File Events:</strong>{{ totalEventsDatasource(datasource.file_on_disk) | compactNumber }}
              </li>
              <li v-if="dataSourceStatus(datasource) === 'fail'">
                <strong>Error message:</strong>
                <code v-if="datasource.error_message"> {{ datasource.error_message }}</code>
              </li>
            </ul>
            <br />
          </v-alert>

          <v-card outlined v-if="percentComplete > 0.1">
            <v-card-title>{{ eventsPerSecond.slice(-1)[0] }} events/s</v-card-title>
            <v-sparkline
              :value="eventsPerSecond"
              :gradient="sparkline.gradient"
              :smooth="sparkline.radius || false"
              :padding="sparkline.padding"
              :line-width="sparkline.width"
              :stroke-linecap="sparkline.lineCap"
              :gradient-direction="sparkline.gradientDirection"
              :fill="sparkline.fill"
              :type="sparkline.type"
              :auto-line-width="sparkline.autoLineWidth"
              auto-draw
            >
            </v-sparkline>
            <v-sheet class="py-4 px-3">
              <v-progress-linear color="light-blue" height="25" :value="percentComplete" rounded>
                {{ percentComplete }}% (complete {{ processingETA() }})
              </v-progress-linear>
            </v-sheet>
          </v-card>
          <v-card v-else outlined class="pa-3"> Waiting for processing to begin.. </v-card>
        </div>
        <v-divider></v-divider>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" text @click="dialogStatus = false"> Close </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-menu v-else offset-y :close-on-content-click="false" content-class="menu-with-gap">
      <template v-slot:activator="{ on }">
        <v-chip v-on="on" :style="getTimelineStyle(timeline)" class="mr-2 mb-3">
          <v-icon v-if="timelineStatus === 'fail'" left color="red"> mdi-alert-circle-outline </v-icon>
          <span class="timeline-name-ellipsis">{{ timeline.name }}</span>

          <span v-if="timelineStatus === 'processing'" class="ml-3">
            <v-progress-circular small indeterminate color="grey" :size="20" :width="2"></v-progress-circular>
          </span>
          <v-avatar
            v-if="timelineStatus === 'ready'"
            right
            style="background-color: rgba(0, 0, 0, 0.1); font-size: 0.55em"
          >
            {{ eventsCount | compactNumber }}
          </v-avatar>
        </v-chip>
      </template>
      <v-sheet flat width="320">
        <v-list>
          <v-dialog v-model="dialogRename" width="600">
            <template v-slot:activator="{ on, attrs }">
              <v-list-item v-bind="attrs" v-on="on">
                <v-list-item-action>
                  <v-icon>mdi-square-edit-outline</v-icon>
                </v-list-item-action>
                <v-list-item-subtitle>Rename timeline</v-list-item-subtitle>
              </v-list-item>
            </template>
            <v-card class="pa-4">
              <h3>Rename timeline</h3>
              <br />
              <v-text-field outlined dense autofocus v-model="newTimelineName" @focus="$event.target.select()">
              </v-text-field>
              <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn color="primary" text @click="dialogRename = false"> Close </v-btn>
                <v-btn color="primary" depressed @click="rename"> Save </v-btn>
              </v-card-actions>
            </v-card>
          </v-dialog>

          <v-list-item @click="$emit('toggle', timeline)" v-if="timelineStatus === 'ready'">
            <v-list-item-action>
              <v-icon v-if="isSelected">mdi-eye-off</v-icon>
              <v-icon v-else>mdi-eye</v-icon>
            </v-list-item-action>
            <v-list-item-subtitle v-if="isSelected">Temporarily disabled</v-list-item-subtitle>
            <v-list-item-subtitle v-else>Re-enable</v-list-item-subtitle>
          </v-list-item>

          <v-dialog v-model="dialogStatus" width="600">
            <template v-slot:activator="{ on, attrs }">
              <v-list-item v-bind="attrs" v-on="on">
                <v-list-item-action>
                  <v-icon>{{ iconStatus }}</v-icon>
                </v-list-item-action>
                <v-list-item-subtitle>Data sources ({{ datasources.length }})</v-list-item-subtitle>
              </v-list-item>
            </template>
            <v-card>
              <v-app-bar flat dense>{{ timeline.name }}</v-app-bar>
              <div class="pa-3">
                <ul style="list-style-type: none">
                  <li><strong>Opensearch index: </strong>{{ timeline.searchindex.index_name }}</li>
                  <li v-if="timelineStatus === 'processing' || timelineStatus === 'ready'">
                    <strong>Number of events: </strong>
                    {{ allIndexedEvents | compactNumber }}
                  </li>
                  <li><strong>Created by: </strong>{{ timeline.user.username }}</li>
                  <li>
                    <strong>Created at: </strong>{{ timeline.created_at | shortDateTime }}
                    <small>({{ timeline.created_at | timeSince }})</small>
                  </li>
                </ul>

                <br />
                {{ datasources.length }} data source(s) in this timeline <br /><br />
                <v-alert
                  v-for="datasource in datasources"
                  :key="datasource.id"
                  colored-border
                  border="left"
                  elevation="1"
                  :color="datasourceStatusColors(datasource)"
                >
                  <ul style="list-style-type: none">
                    <li><strong>Original filename:</strong> {{ datasource.original_filename }}</li>
                    <li><strong>File on disk:</strong> {{ datasource.file_on_disk }}</li>
                    <li><strong>File size:</strong> {{ datasource.file_size | compactBytes }}</li>
                    <li><strong>Provider:</strong> {{ datasource.provider }}</li>
                    <li v-if="datasource.data_label"><strong>Data label:</strong> {{ datasource.data_label }}</li>
                    <li><strong>Status:</strong> {{ dataSourceStatus(datasource) }}</li>
                    <li>
                      <strong>Total File Events:</strong
                      >{{ totalEventsDatasource(datasource.file_on_disk) | compactNumber }}
                    </li>
                    <li v-if="dataSourceStatus(datasource) === 'fail'">
                      <strong>Error message:</strong>
                      <code v-if="datasource.error_message"> {{ datasource.error_message }}</code>
                    </li>
                  </ul>
                  <br />
                </v-alert>
              </div>
              <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn color="primary" text @click="dialogStatus = false"> Close </v-btn>
              </v-card-actions>
            </v-card>
          </v-dialog>
        </v-list>
        <div class="px-4">
          <v-color-picker
            @update:color="updateColor"
            :value="timeline.color"
            :show-swatches="!showCustomColorPicker"
            :swatches="colorPickerSwatches"
            :hide-canvas="!showCustomColorPicker"
            :hide-sliders="!showCustomColorPicker"
            hide-inputs
            mode="hexa"
            dot-size="15"
          ></v-color-picker>
          <v-btn text x-small class="mt-2" @click="showCustomColorPicker = !showCustomColorPicker">
            <span v-if="showCustomColorPicker">Palette</span>
            <span v-else>Custom color</span>
          </v-btn>
        </div>
        <br />
      </v-sheet>
    </v-menu>
  </span>
</template>

<script>
import Vue from 'vue'
import _ from 'lodash'
import dayjs from '@/plugins/dayjs'
import ApiClient from '../../utils/RestApiClient'

const gradients = [
  ['#222'],
  ['#42b3f4'],
  ['red', 'orange', 'yellow'],
  ['purple', 'violet'],
  ['#00c6ff', '#F0F', '#FF0'],
  ['#f72047', '#ffd200', '#1feaea'],
]

export default {
  props: ['timeline', 'eventsCount', 'isSelected', 'isEmptyState'],
  data() {
    return {
      autoRefresh: false,
      allIndexedEvents: 0, // all indexed events from ready and processed datasources
      totalEvents: null,
      dialogStatus: false,
      dialogRename: false,
      datasources: [],
      timelineStatus: null,
      eventsPerSecond: [],
      newTimelineName: [...this.timeline.name],
      sparkline: {
        width: 2,
        radius: 10,
        padding: 8,
        lineCap: 'round',
        gradient: gradients[5],
        gradientDirection: 'bottom',
        gradients,
        fill: false,
        type: 'trend',
        autoDrawDuration: 4000,
        autoLineWidth: false,
      },
      showCustomColorPicker: false,
      colorPickerSwatches: [
        ['#5E75C2', '#BB77C4', '#FD7EAC'],
        ['#FF9987', '#FFC66A', '#F9F871'],
        ['#FFB5BC', '#97D788', '#9BC1AF'],
        ['#FFC7A0', '#FFDF79', '#FFEAEF'],
        ['#DEBBFF', '#9AB0FB', '#CFFBE2'],
      ],
    }
  },
  computed: {
    meta() {
      return this.$store.state.meta
    },
    datasourceErrors() {
      return this.timeline.datasources.filter((datasource) => datasource.error_message)
    },
    datasourcesProcessing() {
      return this.datasources.filter(
        (datasource) =>
          this.dataSourceStatus(datasource) === 'processing' || this.dataSourceStatus(datasource) === 'queueing'
      )
    },
    sketch() {
      return this.$store.state.sketch
    },
    totalEventsToIndex() {
      return this.datasources
        .filter((x) => this.dataSourceStatus(x) === 'processing')
        .map((x) => x.total_file_events)
        .reduce((a, b) => a + b, 0)
    },
    secondsToComplete() {
      return this.totalEventsToIndex / this.avarageEventsPerSecond()
    },
    percentComplete() {
      return Math.floor((this.secondsSinceStart() / this.secondsToComplete) * 100) || 0
    },
    iconStatus() {
      if (this.timelineStatus === 'ready') return 'mdi-information-outline'
      if (this.timelineStatus === 'processing') return 'mdi-circle-slice-7'
      return 'mdi-alert-circle-outline'
    },
  },
  methods: {
    rename() {
      this.dialogRename = false
      this.$emit('save', this.timeline, this.newTimelineName)
    },
    remove() {
      if (confirm('Delete the timeline?')) {
        this.$emit('remove', this.timeline)
      }
    },
    secondsSinceStart() {
      if (!this.datasourcesProcessing.length) {
        return 0
      }
      let start = dayjs.utc(this.datasourcesProcessing[0].updated_at)
      let end = dayjs.utc()
      let diffSeconds = end.diff(start, 'second')
      return diffSeconds
    },
    avarageEventsPerSecond() {
      const sum = this.eventsPerSecond.reduce((a, b) => a + b, 0)
      const avg = sum / this.eventsPerSecond.length || 0
      return Math.floor(avg)
    },
    processingETA() {
      let secondsLeft = this.secondsToComplete - this.secondsSinceStart()
      let eta = dayjs().add(secondsLeft, 'second').fromNow()
      return eta
    },
    // Set debounce to 300ms to limit requests to the server.
    updateColor: _.debounce(function (color) {
      Vue.set(this.timeline, 'color', color.hex.substring(1))
      this.$emit('save', this.timeline)
    }, 300),
    getTimelineStyle(timeline) {
      let backgroundColor = timeline.color
      let textDecoration = 'none'
      let opacity = '50%'
      let p = 100
      if (!backgroundColor.startsWith('#')) {
        backgroundColor = '#' + backgroundColor
      }
      if (this.timelineStatus === 'ready') {
        // Grey out the index if it is not selected.
        if (!this.isSelected) {
          backgroundColor = '#d2d2d2'
          textDecoration = 'line-through'
        } else {
          opacity = '100%'
        }
      } else {
        backgroundColor = '#f5f5f5'
        opacity = '100%'
      }
      let bgColor = 'linear-gradient(90deg, ' + backgroundColor + ' ' + p + '%, #d2d2d2  0%) '
      if (this.$vuetify.theme.dark) {
        return {
          background: bgColor,
          filter: 'grayscale(25%)',
          color: '#333',
        }
      }
      return {
        background: bgColor,
        'text-decoration': textDecoration,
        opacity: opacity,
      }
    },
    fetchData() {
      ApiClient.getSketchTimeline(this.sketch.id, this.timeline.id)
        .then((response) => {
          this.timelineStatus = response.data.objects[0].status[0].status
          this.datasources = response.data.objects[0].datasources
          // This is a naive approach and is misleading if there are multiple
          // datasources importing at the same time, or multiple timelines importing to
          // the same index.
          //
          // TODO: Add datasource ID to all events at import time like we do with
          // timeline ID. This will give us the ability to calculate index progress per
          // datasource, and also enable deletion of datasources from opensearch
          // indices.
          //
          // Tracking in: https://github.com/google/timesketch/issues/2361
          let tmpAllIndexedEvents = this.allIndexedEvents
          this.allIndexedEvents = response.data.meta.lines_indexed
          let deltaEvents = this.allIndexedEvents - tmpAllIndexedEvents

          if (deltaEvents < 10000 && deltaEvents > 0) {
            this.eventsPerSecond.push(Math.floor(deltaEvents / 5))
          }

          if (this.timelineStatus !== 'ready' && this.timelineStatus !== 'fail') {
            this.autoRefresh = true
          } else {
            this.autoRefresh = false
            this.$store.dispatch('updateSketch', this.sketch.id).then(() => {
              if (this.timelineStatus === 'ready') this.$emit('toggle', response.data.objects[0])
            })
          }
        })
        .catch((e) => {
          console.log(e)
        })
    },
    dataSourceStatus(datasource) {
      // Support legacy datasources that don't have a status set.
      if (!datasource.status[0]) {
        return 'ready'
      }

      return datasource.status[0].status
    },

    datasourceStatusColors(datasource) {
      // Support legacy datasources that don't have a status set.
      if (!datasource.status[0]) {
        return 'ready'
      }

      if (datasource.status[0].status === 'ready' || datasource.status === []) {
        return 'success'
      } else if (datasource.status[0].status === 'processing') {
        return 'info'
      } else if (datasource.status[0].status === 'queueing') {
        return 'warning'
      }
      // status = fail
      return 'error'
    },
    totalEventsDatasource(fileOnDisk) {
      return this.datasources.find((x) => x.file_on_disk === fileOnDisk).total_file_events
    },
  },
  created() {
    // TODO: Move to computed
    this.timelineStatus = this.timeline.status[0].status
    this.datasources = this.timeline.datasources

    if (this.timelineStatus === 'processing') {
      this.autoRefresh = true
    } else {
      this.autoRefresh = false
      this.allIndexedEvents = this.meta.stats_per_timeline[this.timeline.id]['count']
    }
    this.newTimelineName = this.timeline.name
  },
  beforeDestroy() {
    clearInterval(this.t)
    this.t = false
  },
  watch: {
    autoRefresh(val) {
      if (val && !this.t) {
        this.t = setInterval(
          function () {
            this.fetchData()
          }.bind(this),
          5000
        )
      } else {
        clearInterval(this.t)
        this.t = false
      }
    },
    timeline() {
      if (this.timeline.datasources.length > this.datasources.length) {
        this.fetchData()
      }
    },
  },
}
</script>

<!-- CSS scoped to this component only -->
<style scoped lang="scss"></style>
