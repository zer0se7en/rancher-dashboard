<script lang="ts">
import Vue, { PropType } from 'vue';
import Application from '@/products/epinio/models/applications.class';
import SortableTable from '@/components/SortableTable/index.vue';
import Checkbox from '@/components/form/Checkbox.vue';
import BadgeState from '@/components/BadgeState.vue';
import ApplicationAction, { APPLICATION_ACTION_TYPE } from '@/products/epinio/models/application-action.class';

import AsyncButton from '@/components/AsyncButton.vue';
import { STATE, DESCRIPTION, SIMPLE_NAME } from '@/config/table-headers';
import { EPINIO_TYPES } from '@/products/epinio/types';

interface Data {
  running: boolean;
  actionHeaders: any[];
  actions: ApplicationAction[]
}

export default Vue.extend<Data, any, any, any>({

  components: {
    SortableTable,
    BadgeState,
    Checkbox,
    AsyncButton
  },

  props: {
    application: {
      type:     Object as PropType<Application>,
      required: true
    },
    source: {
      type: Object as PropType<{
        tarball: string,
        builderImage: string,
      }>,
      required: true
    },
    mode: {
      type:     String,
      required: true
    },
    step: {
      type:     Object as PropType<any>,
      required: true
    }
  },

  async fetch() {
    this.actions.push(await this.$store.dispatch('epinio/create', {
      action:      APPLICATION_ACTION_TYPE.CREATE,
      application: this.application,
      type:        EPINIO_TYPES.APP_ACTION,
      index:       0,
    }));
    this.actions.push(await this.$store.dispatch('epinio/create', {
      action:      APPLICATION_ACTION_TYPE.UPLOAD,
      application: this.application,
      type:        EPINIO_TYPES.APP_ACTION,
      index:       1,
    }));
    this.actions.push(await this.$store.dispatch('epinio/create', {
      action:      APPLICATION_ACTION_TYPE.BUILD,
      application: this.application,
      type:        EPINIO_TYPES.APP_ACTION,
      index:       2,
    }));
    this.actions.push(await this.$store.dispatch('epinio/create', {
      action:      APPLICATION_ACTION_TYPE.DEPLOY,
      application: this.application,
      type:        EPINIO_TYPES.APP_ACTION,
      index:       3,
    }));
  },

  data() {
    return {
      running:       false,
      actionHeaders: [
        {
          ...SIMPLE_NAME,
          sort:     undefined,
          labelKey: 'epinio.applications.steps.progress.table.stage.label',
          width:    200
        },
        {
          ...DESCRIPTION,
          sort:  undefined,
          value: 'description',
          width: 500
        },
        {
          name:     'index',
          labelKey: 'epinio.applications.steps.progress.table.run.label',
          value:    'run',
          sort:     ['index'],
          width:     100,
        },
        {
          ...STATE,
          sort:     undefined,
          labelKey: 'epinio.applications.steps.progress.table.status',
          width:     undefined, // Note - SortableTable will remove the width from NAME col if all columns have width
        },
      ],
      actions: []
    };
  },

  computed: {
    actionsToRun() {
      return this.actions.filter((action: ApplicationAction) => action.run);
    }
  },

  watch: {
    running(neu, prev) {
      if (prev && !neu) {
        Vue.set(this.step, 'ready', true);
      }
    }
  },

  methods: {

    async run(buttonDone: (res: boolean) => void) {
      Vue.set(this, 'running', true);

      const enabledActions = [...this.actionsToRun];

      for (const action of enabledActions) {
        try {
          await action.execute({ source: this.source });
        } catch (err) {
          buttonDone(false);
          Vue.set(this, 'running', false);
          console.error(err);// eslint-disable-line no-console

          return;
        } finally {
          this.$store.dispatch('epinio/find', {
            type: this.application.type,
            id:   `${ this.application.namespace }/${ this.application.name }`,
            opt:  { force: true }
          });
        }
      }
      buttonDone(true);
      Vue.set(this, 'running', false);
    }
  }
});

</script>

<template>
  <div v-if="!$fetchState.pending" class="progress-container">
    <div class="progress">
      <SortableTable
        :rows="actions"
        :headers="actionHeaders"
        :table-actions="false"
        :row-actions="false"
        default-sort-by="index"
        :search="false"
        key-field="key"
      >
        <template #cell:index="{row}">
          <Checkbox v-model="row.run" :disabled="true" />
        </template>
        <template #cell:state="{row}">
          <BadgeState :color="row.stateBackground" :label="row.stateDisplay" />
        </template>
      </SortableTable>
      <div class="run">
        <AsyncButton
          :mode="'run'"
          class="mt-40"
          :disabled="actionsToRun.length === 0"
          @click="run"
        />
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.progress-container {
  display: flex;
  justify-content: center;
  .progress {
    padding: 10px 0;

    .run {
      display: flex;
      justify-content: flex-end;
    }
  }
}

</style>
