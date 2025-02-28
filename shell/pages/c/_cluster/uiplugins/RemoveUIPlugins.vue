<script>
import { CATALOG, UI_PLUGIN } from '@shell/config/types';
import Dialog from '@shell/components/Dialog.vue';
import Checkbox from '@components/Form/Checkbox/Checkbox.vue';
import {
  UI_PLUGIN_NAMESPACE,
  UI_PLUGIN_CHARTS,
  UI_PLUGINS_REPO_NAME,
  UI_PLUGINS_REPO_URL,
  UI_PLUGIN_OPERATOR_CRD_CHART_NAME,
} from '@shell/config/uiplugins';

export default {
  components: {
    Checkbox,
    Dialog,
  },

  async fetch() {
    if (this.$store.getters['management/schemaFor'](CATALOG.CLUSTER_REPO)) {
      const repos = await this.$store.dispatch('management/findAll', { type: CATALOG.CLUSTER_REPO, opt: { force: true } });

      this.defaultRepo = repos.find((r) => r.name === UI_PLUGINS_REPO_NAME && r.spec.gitRepo === UI_PLUGINS_REPO_URL);
    }

    if (this.$store.getters['management/schemaFor'](UI_PLUGIN)) {
      const plugins = await this.$store.dispatch('management/findAll', { type: UI_PLUGIN });

      // Are there any plugins installed?
      this.hasPluginsInstalled = (plugins || []).length > 0;
      this.removeCRD = !this.hasPluginsInstalled;
    }
  },

  data() {
    return {
      errors:              [],
      defaultRepo:         undefined,
      removeRepo:          false,
      removeCRD:           true,
      hasPluginsInstalled: false,
    };
  },

  methods: {
    async removeChart(name) {
      const apps = await this.$store.dispatch('management/findAll', { type: CATALOG.APP });
      const found = apps.find((app) => {
        return app.namespace === UI_PLUGIN_NAMESPACE && app.name === name;
      });

      if (found) {
        return found.remove();
      }

      // Return rejected promise - could not find the required chart
      return Promise.reject(new Error(`Could not find chart §{ name }`));
    },

    showDialog() {
      this.removeRepo = !!this.defaultRepo;
      this.$modal.show('confirm-uiplugins-remove');
    },

    async doRemove(btnCb) {
      this.errors = [];

      // Remove the charts in the reverse order that we install them in
      let uninstall = [...UI_PLUGIN_CHARTS].reverse();

      if (!this.removeCRD) {
        // User does not want to uninstall the CRD, so remove the chart
        uninstall = uninstall.filter((chart) => chart !== UI_PLUGIN_OPERATOR_CRD_CHART_NAME);
      }

      for (let i = 0; i < uninstall.length; i++) {
        const chart = uninstall[i];

        try {
          await this.removeChart(chart);
        } catch (e) {
          this.errors.push(e.message);
        }
      }

      if (this.removeRepo && this.defaultRepo) {
        try {
          await this.defaultRepo.remove();
        } catch (e) {
          this.errors.push(e.message);
        }
      }

      this.$store.dispatch('management/forgetType', UI_PLUGIN);

      await new Promise((resolve) => setTimeout(resolve, 5000));

      this.$emit('done');

      btnCb(true);
    },
  }
};
</script>
<template>
  <Dialog
    name="confirm-uiplugins-remove"
    :title="t('plugins.setup.remove.title')"
    mode="disable"
    data-testid="disable-ext-modal"
    @okay="doRemove"
  >
    <template>
      <p>
        {{ t('plugins.setup.remove.prompt') }}
      </p>
      <div
        v-if="!!defaultRepo"
        class="mt-20"
      >
        <Checkbox
          v-model="removeRepo"
          :primary="true"
          label-key="plugins.setup.remove.registry.title"
          data-testid="disable-ext-modal-remove-repo"
        />
        <div class="checkbox-info">
          {{ t('plugins.setup.remove.registry.prompt') }}
        </div>
      </div>
      <div
        v-if="hasPluginsInstalled"
        class="mt-20"
      >
        <Checkbox
          v-model="removeCRD"
          :primary="true"
          label-key="plugins.setup.remove.crd.title"
        />
        <div class="checkbox-info">
          {{ t('plugins.setup.remove.crd.prompt') }}
        </div>
      </div>
    </template>
  </Dialog>
</template>
<style lang="scss" scoped>
  .enable-plugin-support {
    font-size: 14px;
    margin-top: 20px;
  }

  .plugin-setup-error {
    font-size: 14px;
    color: var(--error);
    margin: 10px 0 0 0;
  }

  .checkbox-info {
    margin-left: 20px;
    opacity: 0.7;
  }
</style>
