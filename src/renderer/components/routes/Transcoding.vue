<template>
  <page>
    <template slot="title">
      {{ $t('Transcoding.Transcoding') }}
    </template>

    <template slot="body">
      <b-table :data="transcodings">
        <template slot-scope="props">
          <b-table-column label="Id">
            {{ props.row.id }}
          </b-table-column>
          <b-table-column :label="$t('Transcoding.Item')">
            <item-link :itemId="props.row.itemId"></item-link>
          </b-table-column>
          <b-table-column :label="$t('Transcoding.Codec')">
            {{ props.row.codec }}
          </b-table-column>
          <b-table-column :label="$t('Transcoding.Width')">
            {{ props.row.width }}
          </b-table-column>
          <b-table-column :label="$t('Transcoding.Height')">
            {{ props.row.height }}
          </b-table-column>
          <b-table-column :label="$t('Transcoding.Progress')">
            {{ props.row.progress }}
          </b-table-column>
          <b-table-column>
            <a @click="deleteJob" :data-id="props.row.id">{{ $t('Transcoding.delete') }}</a>
          </b-table-column>
        </template>
      </b-table>
      <button v-if="$store.state.transcoding" class="button" @click="stop">{{ $t('Transcoding.Stop') }}</button>
      <button v-else class="button" @click="start">{{ $t('Transcoding.Start') }}</button>
      <template v-if="isDevelopment">
        <b-field label="H.264 CRF" :message="$t('Transcoding.h264CrfMessage')">
          <b-input v-model="h264Crf" type="number" min="18" max="28">
          </b-input>
        </b-field>
        <b-field :label="$t('Transcoding.h264EncodingSpeed')" :message="$t('Transcoding.h264EncodingSpeedMessage')">
          <b-select v-model="h264Preset">
            <option value="veryslow">Very slow</option>
            <option value="slower">Slower</option>
            <option value="slow">Slow</option>
            <option value="medium">Medium</option>
            <option value="fast">Fast</option>
            <option value="faster">Faster</option>
            <option value="veryfast">Very fast</option>
          </b-select>
        </b-field>
        <b-field label="VP9 CRF" :message="$t('Transcoding.vp9CrfMessage')">
          <b-input v-model="vp9Crf" type="number" min="0" max="63">
          </b-input>
        </b-field>
        <b-field :label="$t('Transcoding.vp9EncodingSpeed')" :message="$t('Transcoding.vp9EncodingSpeedMessage')">
          <b-select v-model="vp9Speed">
            <option value="0">0</option>
            <option value="1">1</option>
            <option value="2">2</option>
            <option value="3">3</option>
          </b-select>
        </b-field>
      </template>
    </template>
  </page>
</template>

<script lang="ts">
  import Vue from 'vue'
  import Page from '../Page.vue'
  import ItemLink from '../ItemLink.vue'
  import setTitle from '../../../lib/setTitle'

  export default Vue.extend({
    name: 'transcoding',
    components: {
      Page,
      ItemLink,
    },
    data() {
      return {
        isDevelopment: this.$settings.get('development'),
        h264Crf: this.$settings.get('h264.crf'),
        h264Preset: this.$settings.get('h264.preset'),
        vp9Crf: this.$settings.get('vp9.crf'),
        vp9Speed: this.$settings.get('vp9.speed'),
      }
    },
    async created() {
      setTitle(this.$t('Transcoding.Transcoding'))
    },
    computed: {
      transcodings() {
        return this.$store.state.transcodings
      }
    },
    methods: {
      deleteJob(event: any) {
        this.$root.$emit('transcodeRemoveJob', parseInt(event.target.dataset.id))
      },
      start(event: any) {
        this.$root.$emit('transcodeStart')
      },
      stop(event: any) {
        this.$root.$emit('transcodeStop')
      },
    },
    watch: {
      h264Crf() {
        this.$settings.set('h264.crf', this.h264Crf)
      },
      h264Preset() {
        this.$settings.set('h264.preset', this.h264Preset)
      },
      vp9Crf() {
        this.$settings.set('vp9.crf', this.vp9Crf)
      },
      vp9Speed() {
        this.$settings.set('vp9.speed', this.vp9Speed)
      },
    },
  })
</script>
