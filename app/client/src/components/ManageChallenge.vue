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

Name:     components/ManageChallenge.vue
Purpose:  Dispaly, edit, add, and delete challenge from the admin console
Methods:
  * Allows the challenge to be added
  * Allows the challenge to be deleted
  * Allows the challenge to be eited
  * Display the list of chanllenges in a table

## -->

<template>
  <q-page flat class="">
    <!-- -------------------- Main Content -------------------- -->
    <q-table
      flat wrap-cells binary-state-sort
      color="secondary"
      row-key="uuid"
      :data="projectList"
      :columns="columns"
      :filter="filter"
      :loading="loading"
      :pagination.sync="pagination"
    >
      <template v-slot:top-left>
        <q-btn
          round
          icon="add" color="accent"
          @click="dialog = true; dialogOption = 'add'"
        />
      </template>

      <template v-slot:top-right>
        <q-toolbar>
          <q-btn
            dense flat round
            icon="menu"
            class="q-mr-xs"
          >
            <q-menu dense>
              <q-list
                v-for="(keyword, index) in popkeywords"
                :key="index"
                style="min-width: 100px"
              >
                <q-item clickable v-close-popup dense>
                  <q-item-section @click="filter = keyword.value">
                    {{ keyword.label }}
                  </q-item-section>
                </q-item>
                <q-separator />
              </q-list>
            </q-menu>
          </q-btn>

          <q-space />

          <q-input
            dense
            debounce="300" color="primary" placeholder="Search"
            v-model="filter"
          >
            <template v-slot:append>
              <q-icon
                v-if="filter === ''"
                name="search"
              />
              <q-icon
                v-else
                name="clear"
                class="cursor-pointer"
                @click="filter = ''"
              />
            </template>
          </q-input>
        </q-toolbar>
      </template>

      <template v-slot:header="props">
        <q-tr :props="props">
          <q-th
            v-for="col in props.cols"
            :key="col.name"
            :props="props"
            style="font-size: 18px; font-weight: normal;"
          >
            {{ col.label }}
          </q-th>
        </q-tr>
      </template>

      <template v-slot:body="props">
        <q-tr :props="props">

          <q-td
            key="keywords"
            :props="props"
          >
            <div
              hidden
              align="left"
            >
              {{ props.row.keywords }}
            </div>
          </q-td>

          <q-td
            key="name"
            :props="props"
            style="width: 300px;"
          >
            <div align="left">
              {{ props.row.challenge }}
            </div>
          </q-td>

          <q-td
            key="alias"
            :props="props"
          >
            <div align="left">
              {{ props.row.alias }}
            </div>
          </q-td>

          <q-td
            key="uuid"
            :props="props"
          >
            <div align="center">
              {{ props.row.uuid }}
            </div>
          </q-td>

          <q-td
            key="icons"
            :props="props"
            style="width: 100px;"
          >
            <q-btn
              dense round flat
              color="secondary" icon="edit"
              @click="editProject(props.row.uuid)"
            />

            <q-btn
              dense round flat
              color="secondary" icon="delete"
              @click="deleteChallenge(props.row.uuid, props.row.alias)"
            />
          </q-td>

        </q-tr>
      </template>

    </q-table>

    <q-dialog
      persistent maximized
      transition-show="slide-up" transition-hide="slide-down"
      v-model="dialog"
    >
      <q-card>

        <q-card-section
          v-if="dialogOption === 'add'"
        >
          <addChallenge
            @added="updateProjectsAndClose"
            @close="dialog = false"
          />
        </q-card-section>

        <q-card-section
          v-if="dialogOption === 'edit'"
        >
          <br>
          <popUpChallenge
            :challengeId="uuid"
            :mode="dialogOption"
            @added="updateProjects"
            @close="dialog = false"
          />
        </q-card-section>

      </q-card>
    </q-dialog>

  </q-page>
</template>

<script>
import firebase from 'firebase/app'
import 'firebase/firestore'

import productionDb, { proAppCall } from '../firebase/init_production'
import testingDb, { testAppCall } from '../firebase/init_testing'

import addChallenge from '../components/SubmitChallengeAdminConsole'
import popUpChallenge from '../components/EditAndPreviewChallenge'

export default {
  components: {
    addChallenge,
    popUpChallenge
  },
  async created () {
    try {
      await this.loadFireRefs()
      await this.loadChallenges()
      await this.loadConfig()
    } catch (error) {
      throw new Error(error)
    }
  },
  data () {
    return {
      db: null, // <Object>: firebase object referencing the database
      cloudFunctions: null, // <Object>: firebase ref to cloud functions
      uuid: '', // <String>: uid of the challenge to be edited
      // popkeywords <Array<Object>>: list of keywords converted to object
      //                              with label and value
      popkeywords: [],
      dialog: false, // <Boolean>: flag to invoke dialog
      dialogOption: '', // <String>: mode of the dialog
      projectList: [], // <Array<Object>>: list of challenges from ToC
      uuidList: [], // <Array<String>>: list of uid from ToC
      filter: '', // <String>: value of the search
      loading: true, // <Boolean>: flag for page loading
      pagination: { // <Object>: pagination object for the table
        sortBy: 'name', // <String>: name of the column to be sorted
        rowsPerPage: 50 // <Integer>: number of items to be listed per page
      },
      // columns <Array<Object>>: column layout of the display table
      columns: [
        {
          name: 'keywords',
          label: '',
          field: row => row.keywords
        },
        {
          name: 'name',
          required: true,
          align: 'center',
          label: 'Challenge Name',
          field: row => row.challenge,
          format: val => `${val}`,
          sort: (a, b) => {
            if (a.trim() < b.trim()) {
              return -1
            } else if (a.trim() > b.trim()) {
              return 1
            } else {
              return 0
            }
          },
          sortable: true
        },
        {
          name: 'alias',
          required: true,
          align: 'center',
          label: 'Alias',
          field: row => row.alias || '',
          format: val => `${val}`,
          sortable: true
        },
        {
          name: 'uuid',
          required: true,
          label: 'UUID',
          align: 'center',
          field: row => row.uuid,
          format: val => `${val}`,
          sortable: true
        },
        {
          name: 'icons',
          align: 'center',
          label: ''
        }
      ]
    }
  },
  methods: {
    loadFireRefs: async function () {
      /**
       * load firebase database reference
       * load firebase storage reference (if applicable)
       * load firebase cloud functions reference (if applicable)
       * @param {void}
       * @return {Promise<Boolean>}
       */

      if (this.$q.localStorage.has('boundless_db')) {
        let sessionDb = this.$q.localStorage.getItem('boundless_db')

        if (sessionDb === 'testing') {
          this.db = testingDb
          this.cloudFunctions = testAppCall.httpsCallable('appCall')
        } else {
          this.db = productionDb
          this.cloudFunctions = proAppCall.httpsCallable('appCall')
        }

        return true
      } else {
        try {
          let doc = await productionDb.collection('config').doc('project').get()

          if (doc.exists) {
            if (doc.data().db === 'testing') {
              this.db = testingDb
              this.cloudFunctions = testAppCall.httpsCallable('appCall')
              this.$q.localStorage.set('boundless_db', 'testing')
            } else {
              this.db = productionDb
              this.cloudFunctions = proAppCall.httpsCallable('appCall')
              this.$q.localStorage.set('boundless_db', 'production')
            }

            return true
          } else {
            let msg = '"/config/project" path does not exists!'

            throw new Error(msg)
          }
        } catch (error) {
          this.db = productionDb
          this.cloudFunctions = proAppCall.httpsCallable('appCall')
          this.$q.localStorage.set('boundless_db', 'production')

          return false
        }
      }
    },
    loadConfig: async function () {
      /**
       * load keywords into this.popkeywords by converting to fit q-option
       * @param {void}
       * @return {Promise<Boolean>}
       */

      if (this.$q.sessionStorage.has('boundless_config')) {
        let cachedConfig = this.$q.sessionStorage.getItem('boundless_config')

        this.pagination.rowsPerPage = cachedConfig.pagination
      }

      try {
        let doc = await this.db.collection('config').doc('project').get()

        if (doc.exists) {
          let data = doc.data()

          for (let key in data['keywords']) {
            this.popkeywords.push({
              label: key,
              value: data['keywords'][key]
            })
          }

          return true
        } else {
          throw new Error('"config/project" not found!')
        }
      } catch (error) {
        return false
      }
    },
    loadChallenges: async function () {
      /**
       * load all the challenges from the ToC for the admin console
       * @param {void}
       * @return {Promise<Boolean>}
       */

      try {
        let doc = await this.db.collection('challenges').doc('ToC').get()

        if (doc.exists) {
          let tocProjectData = doc.data()

          for (let project in tocProjectData) {
            if (project !== 'alias') {
              this.projectList.push(tocProjectData[project])
              this.uuidList.push(project)
            }
          }
        } else {
          throw new Error('No -ToC- document!')
        }

        setTimeout(() => {
          this.loading = false

          return true
        }, 300)
      } catch (error) {
        this.loading = false

        return false
      }
    },
    deleteChallenge: async function (entry, removedAlias) {
      /**
       * deletes project from the database and stroage;
       * notifies the user of the status when compeleted
       * @param{String} entry: uid of the project to be removed
       * @param {String} removedAlias: alias of the project to be removed
       * @return {void}
       */

      this.$q.dialog({
        title: 'Confirmation to Delete',
        message: `Delete ${entry}?`,
        ok: true,
        cancel: true
      })
        .onOk(async () => {
          if (this.projectList.length < 1) {
            this.$q.dialog({
              title: 'Error',
              message: 'Nothing to remove!'
            })
          } else {
            if (this.uuidList.includes(entry)) {
              try {
                await this.db.collection('challenges').doc(entry).delete()

                this.$q.notify({
                  type: 'positive',
                  message: 'Deleted sucessfully!'
                })

                let updates = {}
                updates[entry] = firebase.firestore.FieldValue.delete()

                if (typeof removedAlias !== 'undefined') {
                  if (removedAlias !== '') {
                    updates[`alias.${removedAlias}`] = firebase.firestore.FieldValue.delete()
                  }
                }

                await this.db.collection('challenges').doc('ToC')
                  .update(updates)

                let tmpProjectList = []

                this.projectList.forEach(project => {
                  if (project.uuid !== entry) {
                    tmpProjectList.push(project)
                  }
                })

                this.projectList = tmpProjectList

                // delete the storage dir from the storage
                await this.cloudFunctions({ folder: `challenges/${entry}` })
              } catch (error) {
              }
            } else {
              this.$q.dialog({
                title: 'Error',
                message: 'UUID does not exist in the database.'
              })
            }
          }
        })
        .onCancel(() => {
        })
    },
    updateProjectsAndClose: function () {
      /**
       * helper function to update the list of projects and close the dialog
       * @param {void}
       * @return {void}
       */

      this.loading = true

      this.projectList = []
      this.uuidList = []

      this.loadChallenges()

      this.dialog = false
    },
    updateProjects: function () {
      /**
       * helper function to refetch project list
       * @param {void}
       * @return {void}
       */

      this.loading = true

      this.projectList = []
      this.uuidList = []

      this.loadChallenges()
    },
    editProject: function (entry) {
      /**
       * helper function to dialog to invoke 'edit'
       * @param {String} entry: uid of the project
       * @returns {void}
       */

      this.dialogOption = 'edit'
      this.uuid = entry

      setTimeout(() => {
        this.dialog = true
      }, 200)
    }
  }
}
</script>

<style lang="stylus">

</style>
