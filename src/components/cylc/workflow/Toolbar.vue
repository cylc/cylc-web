<!--
Copyright (C) NIWA & British Crown (Met Office) & Contributors.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<template>
  <v-app-bar
    app
    id="core-app-bar"
    dense
    flat
    class="c-toolbar"
  >
    <!-- TODO: duplicated in workflow/Toolbar.vue and cylc/Toolbar.vue -->
    <!-- burger button for mobile -->
    <v-btn
      v-if="responsive"
      dark
      icon
      @click.stop="onClickBtn"
      class="default v-btn--simple"
      id="toggle-drawer"
    >
      <v-icon>{{ svgPaths.list }}</v-icon>
    </v-btn>
    <!-- title -->
    <v-toolbar-title
      class="tertiary--text font-weight-light"
    >
      <span class="c-toolbar-title">{{ title }}</span>
    </v-toolbar-title>

    <!-- control bar elements displayed only when there is a current workflow in the store -->
    <template v-if="currentWorkflow">
      <v-icon
        id="workflow-mutate-button"
        color="#5E5E5E"
        v-cylc-object="currentWorkflow.id"
      >
        {{ svgPaths.menu }}
      </v-icon>

      <v-icon
        id="workflow-release-hold-button"
        color="#5E5E5E"
        :disabled="!enabled.holdToggle"
        @click="onClickReleaseHold"
      >
        {{ isHeld ? svgPaths.run : svgPaths.hold }}
      </v-icon>

      <v-icon
        id="workflow-stop-button"
        color="#5E5E5E"
        :disabled="!enabled.stopToggle"
        @click="onClickStop"
      >
        {{ svgPaths.stop }}
      </v-icon>

      <!-- TODO: add workflow latest message -->
      <span></span>

      <v-spacer />

      <v-menu
        offset-y
        v-if="$route.name === 'workflow'"
      >
        <template v-slot:activator="{ on }">
          <a class="add-view" v-on="on">
            {{ $t('Toolbar.addView') }} <v-icon color="#5995EB">{{ svgPaths.add }}</v-icon>
          </a>
        </template>
        <v-list class="pa-0">
          <v-list-item
            class="py-0 px-8 ma-0"
            @click="$listeners['add-tree']"
            id="toolbar-add-tree-view"
          >
            <v-list-item-title><v-icon>{{ svgPaths.tree }}</v-icon> Tree</v-list-item-title>
          </v-list-item>
          <v-list-item
            class="py-0 px-8 ma-0"
            @click="$listeners['add-mutations']"
            id="toolbar-add-mutations-view"
          >
            <v-list-item-title><v-icon>{{ svgPaths.mutations }}</v-icon> Mutations</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
    </template>

    <!-- displayed only when extended===true -->
    <template v-slot:extension v-if="extended">
      <span style="margin-left: 260px;">
        <a @click="onClickPause">
          <v-icon color="#5E5E5E">{{ svgPaths.hold }}</v-icon>
        </a>

        <a @click="onClickStop">
          <v-icon color="#5E5E5E">{{ svgPaths.stop }}</v-icon>
        </a>

        <span>Other controls added in the future</span>
      </span>
    </template>
  </v-app-bar>
</template>

<script>
import { mapGetters, mapState } from 'vuex'
import toolbar from '@/mixins/toolbar'
import WorkflowState from '@/model/WorkflowState.model'
import {
  mdiViewList,
  mdiPlay,
  mdiPause,
  mdiStop,
  mdiPlusCircle,
  mdiFileTree,
  mdiAppleKeyboardCommand,
  mdiMicrosoftXboxControllerMenu
} from '@mdi/js'

import {
  mutationStatus
} from '@/utils/aotf'

export default {
  name: 'Toolbar',

  mixins: [
    toolbar
  ],

  data: () => ({
    extended: false,
    // FIXME: remove local state once we have this data in the workflow - https://github.com/cylc/cylc-ui/issues/221
    svgPaths: {
      list: mdiViewList,
      hold: mdiPause,
      run: mdiPlay,
      stop: mdiStop,
      tree: mdiFileTree,
      mutations: mdiAppleKeyboardCommand,
      add: mdiPlusCircle,
      menu: mdiMicrosoftXboxControllerMenu
    },
    expecting: {
      // store state from mutations in order to compute the "enabled" attrs
      held: null,
      stop: null
    }
  }),

  computed: {
    ...mapState('app', ['title']),
    ...mapGetters('workflows', ['currentWorkflow']),
    isHeld () {
      return (
        this.currentWorkflow &&
        this.currentWorkflow.status === WorkflowState.HELD.name
      )
    },
    isStopped () {
      return (
        !this.currentWorkflow ||
        this.currentWorkflow.status === WorkflowState.STOPPED.name
      )
    },
    enabled () {
      // object holding the states of controls that are supposed to be enabled
      // NOTE: this is a temporary solution until we are able to subscribe to
      // mutations to tell when they have completed
      return {
        holdToggle: (
          // the play/pause button
          !this.isStopped &&
          !this.expecting.stop &&
          this.currentWorkflow.status !== WorkflowState.STOPPING.name &&
          (
            this.expecting.held === null ||
            this.expecting.held === this.currentWorkflow.isHeld
          )
        ),
        stopToggle: (
          // the stop button
          !this.isStopped &&
          (
            this.expecting.stop === null ||
            this.expecting.stop === this.currentWorkflow.isStopped
          )
        )
      }
    }
  },

  watch: {
    isHeld () {
      this.expecting.held = null
    },
    isStopped () {
      this.expecting.stop = null
    }
  },

  methods: {
    onClickReleaseHold () {
      const ret = this.$workflowService.mutate(
        this.isHeld ? 'release' : 'hold',
        this.currentWorkflow.id
      )
      if (ret[0] === mutationStatus.SUCCEEDED) {
        this.expecting.held = !this.isHeld
      }
    },
    async onClickStop () {
      const ret = this.$workflowService.mutate(
        'stop',
        this.currentWorkflow.id
      )
      if (ret[0] === mutationStatus.SUCCEEDED) {
        this.expecting.stop = WorkflowState.STOPPING
      }
    },
    toggleExtended () {
      this.extended = !this.extended
    }
  }
}
</script>
