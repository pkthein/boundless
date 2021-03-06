<!-- ##
## Copyright (c) 2019 Wind River Systems, Inc.
##
## Licensed under the Apache License, Version 2.0 (the "License");
## you may not use this file except in compliance with the License.
## You may obtain a copy of the License at:
##       http://www.apache.org/licenses/LICENSE-2.0
## Unless required by applicable law or agreed to in writing, software  distributed
## under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES
## OR CONDITIONS OF ANY KIND, either express or implied.

Name:     components/Markdown.vue
Purpose:  Allows the editing of the right panel body of about page with
          markdown format
Methods:
  * Custmize right panel passage with markdown

## -->

<template>
  <div class="q-pa-md">

    <q-splitter class="q-gutter-sm" v-model="splitterModel" :limits="[30, 70]" >

      <template v-slot:before>
        <div class="q-pa-md">
          <q-input
            autogrow outlined dense
            debounce="300" type="textarea"
            v-model="generalConfig.data"
          />
        </div>

        <div class="q-px-md q-gutter-md">
          <q-btn outline label="Background Color">
            <q-popup-proxy transition-show="scale" transition-hide="scale">
              <q-color flat v-model="generalConfig.bgColor" />

              <q-btn class="float-right" flat v-close-popup >Done</q-btn>
            </q-popup-proxy>
          </q-btn>

          <q-btn outline label="Text Color">
            <q-popup-proxy transition-show="scale" transition-hide="scale">
              <q-color flat v-model="generalConfig.txtColor" />

              <q-btn class="float-right" flat v-close-popup >Done</q-btn>
            </q-popup-proxy>
          </q-btn>

          <q-btn
            outline v-close-popup
            class="float-right" color="green"
            @click="submitMarkdown"
          >
            Submit
          </q-btn>

          <q-btn flat v-close-popup class="float-right" >
            Cancel
          </q-btn>

        </div>
      </template>

      <template v-slot:separator>
        <q-avatar
          color="info"
          text-color="white"
          size="40px" icon="drag_indicator"
        />
      </template>

      <template v-slot:after>
        <div
          class="q-pa-md"
          :style="{
            backgroundColor: generalConfig.bgColor,
            color: generalConfig.txtColor
          }"
        >
          <MarkdownTranslator
            :storage="storage"
            :data="generalConfig.data"
          />
        </div>
      </template>

    </q-splitter>

  </div>
</template>

<script>
import MarkdownTranslator from './MarkdownTranslator'

export default {
  components: {
    MarkdownTranslator
  },
  props: {
    db: Object,
    storage: Object
  },
  created () {
    // load config information from the session storage
    if (this.$q.sessionStorage.has('boundless_config')) {
      let cachedConfig = this.$q.sessionStorage.getItem('boundless_config')
      let oldPlaceHolder = this.generalConfig

      this.generalConfig = cachedConfig.generalConfig || this.generalConfig

      for (let field in oldPlaceHolder) {
        this.generalConfig[field] = this.generalConfig[field] ||
          oldPlaceHolder[field]
      }

      this.generalConfig = this.deepObjCopy(this.generalConfig)
    }
  },
  data () {
    return {
      // generalConfig <Object>: record of the general config object associated
      //                         to the right panel of about page
      generalConfig: {
        // data <String>: string to be displayed on the right about panel
        data: ':::\nPut your markdown here\n\nClassic markup: :wink: :joy: :cry: :angel: :heart: :beers: :laughing: :yum:\n\nShortcuts (emoticons): :-) :-( 8-) ;)\n:::\nMax fixed size image: 200x200; responsive\n![Minion](https://octodex.github.com/images/minion.png =200x200)\n\n# Best of all, the box autogrows!\n',
        // bgColor <String>: hex string encoding color of the background
        bgColor: '#000000',
        txtColor: '#ffffff' // <String>: hex string encoding color of the text
      },
      splitterModel: 50 // <Integer>: page splitter
    }
  },
  methods: {
    submitMarkdown: async function () {
      /**
       * submit handler of the component
       * @param {void}
       * @return {Promise<Boolean>}
       */

      let cachedConfig = this.$q.sessionStorage.getItem('boundless_config')
      cachedConfig.generalConfig = this.generalConfig

      this.$q.sessionStorage.set('boundless_config', cachedConfig)

      try {
        await this.db.collection('config').doc('project').update({
          generalConfig: this.generalConfig
        })

        this.$q.notify({
          color: 'green',
          message: '<div align="center">Sucessful!<div>',
          html: true,
          timeout: 750
        })

        return true
      } catch (error) {
        return false
      }
    },
    deepObjCopy: function (aObject) {
      /**
       * https://stackoverflow.com/questions/4459928/how-to-deep-clone-in-javascript/34624648#34624648
       * creates a deep copy of the input
       * @param {Object} aObject: the object to be cloned
       * @return {Object}
       */

      if (!aObject) {
        return aObject
      }

      let v
      let bObject = Array.isArray(aObject) ? [] : {}
      for (const k in aObject) {
        v = aObject[k]
        bObject[k] = (typeof v === 'object') ? this.deepObjCopy(v) : v
      }

      return bObject
    }
  }
}
</script>

<style lang="stylus">

</style>
