<script lang="ts">
import Vue, { PropType } from 'vue';
import Application from '../../models/applications';

import { EpinioCompRecord, EpinioConfiguration, EpinioService, EPINIO_TYPES } from '../../types';
import { sortBy } from '@shell/utils/sort';
import LabeledSelect from '@shell/components/form/LabeledSelect.vue';
import { _VIEW } from '@shell/config/query-params';
import { EPINIO_APP_MANIFEST } from './AppSource.vue';

export interface EpinioAppBindings {
  configurations: string[],
  services: EpinioService[],
}

interface Data {
  values: {
    configurations: string[],
    services: string[],
  }
}

export default Vue.extend<Data, EpinioCompRecord, EpinioCompRecord, EpinioCompRecord>({
  components: { LabeledSelect },

  props: {
    initialApplication: {
      type:    Object as PropType<Application>,
      default: () => ({}),
    },
    application: {
      type:     Object as PropType<Application>,
      required: true
    },
    mode: {
      type:     String,
      required: true
    },
  },

  async fetch() {
    await Promise.all([
      this.$store.dispatch('epinio/findAll', { type: EPINIO_TYPES.CONFIGURATION }),
      this.$store.dispatch('epinio/findAll', { type: EPINIO_TYPES.SERVICE_INSTANCE }),
    ]);
  },

  data() {
    return {
      values: {
        configurations: [],
        services:       []
      }
    };
  },

  computed: {
    configurations() {
      const list = this.namespacedConfigurations
        .filter((s: EpinioConfiguration) => !s.isServiceRelated)
        .map((s: EpinioConfiguration) => ({
          label: s.metadata.name,
          value: s.metadata.name,
        }));

      return sortBy(list, 'label');
    },

    namespacedServices() {
      return this.$store.getters['epinio/all'](EPINIO_TYPES.SERVICE_INSTANCE)
        .filter((s: EpinioService) => s.metadata.namespace === this.application.metadata.namespace);
    },

    namespacedConfigurations() {
      return (this.$store.getters['epinio/all'](EPINIO_TYPES.CONFIGURATION) || [])
        .filter((s: EpinioService) => s.metadata.namespace === this.application.metadata.namespace);
    },

    services() {
      const list = this.namespacedServices.map((s: EpinioService) => ({
        label: `${ s.metadata.name } (${ s.catalog_service })`,
        value: s,
      }));

      return sortBy(list, 'label');
    },

    noConfigs() {
      return !this.$fetchState.pending && !this.configurations.length;
    },

    hasConfigs() {
      return !this.$fetchState.pending && !!this.configurations.length;
    },

    noServices() {
      return !this.$fetchState.pending && !this.services.length;
    },

    hasServices() {
      return !this.$fetchState.pending && !!this.services.length;
    },

    isView() {
      return this.mode === _VIEW;
    },

    isFromManifest() {
      return this.$route.query.from === EPINIO_APP_MANIFEST;
    }
  },

  watch: {
    values: {
      deep: true,
      handler() {
        this.$emit('change', this.values);
      }
    },

    noConfigs(neu, old) {
      if (neu && this.values.configurations?.length) {
        // Selected configurations are no longer valid
        this.values.configurations = [];
      }
    },

    noServices(neu, old) {
      if (neu && this.values.services?.length) {
        // Selected services are no longer valid
        this.values.services = [];
      }
    },

    hasConfigs(neu, old) {
      if (!old && neu) {
        if (!!this.initialApplication?.configuration?.configurations) {
          // Filter out any we don't know about
          this.values.configurations = this.initialApplication.baseConfigurationsNames?.filter((cc: string) => this.configurations.find((c: any) => c.value === cc)) || [];
        }

        if (this.isFromManifest) {
          this.values.configurations = this.namespacedConfigurations
            .filter((nc: any) => this.application.configuration.configurations.includes(nc.metadata.name) && !nc.isServiceRelated)
            .map(({ name }: { name: string }) => name) || [];
        }
      }
    },

    hasServices(neu, old) {
      if (!old && neu) {
        if (!!this.initialApplication.serviceConfigurationsNames) {
          this.values.services = (this.initialApplication.services || []);
        }
      }

      if (this.isFromManifest) {
        const configurations = this.namespacedConfigurations
          .filter((nc: any) => this.application.configuration.configurations.includes(nc.metadata.name) && nc.isServiceRelated) || [];

        this.values.services = this.services
          .filter((s: any) => configurations.some((d: any) => s.value.metadata.name === d.configuration.origin))
          .map((elem: any) => elem.value);
      }
    }
  },
});

</script>

<template>
  <div>
    <div class="col span-6">
      <LabeledSelect
        v-model="values.configurations"
        data-testid="epinio_app-configuration_configurations"
        :loading="$fetchState.pending"
        :disabled="$fetchState.pending || noConfigs || isView"
        :options="configurations"
        :searchable="true"
        :mode="mode"
        :multiple="true"
        :label="t('typeLabel.configurations', { count: 2})"
        :placeholder="noConfigs ? t('epinio.applications.steps.configurations.configurations.select.placeholderNoOptions') : t('epinio.applications.steps.configurations.configurations.select.placeholderWithOptions')"
      />
    </div>
    <div class="spacer" />
    <div class="col span-6">
      <LabeledSelect
        v-model="values.services"
        data-testid="epinio_app-configuration_services"
        :loading="$fetchState.pending"
        :disabled="$fetchState.pending || noServices || isView"
        :options="services"
        :searchable="true"
        :mode="mode"
        :multiple="true"
        :label="t('epinio.applications.steps.configurations.services.select.label')"
        :placeholder="noServices ? t('epinio.applications.steps.configurations.services.select.placeholderNoOptions') : t('epinio.applications.steps.configurations.services.select.placeholderWithOptions')"
      />
    </div>
  </div>
</template>

<style lang='scss' scoped>
.labeled-select {
  min-height: 79px;
}
</style>
