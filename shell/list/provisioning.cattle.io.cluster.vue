<script>
import { Banner } from '@components/Banner';
import ResourceTable from '@shell/components/ResourceTable';
import Masthead from '@shell/components/ResourceList/Masthead';
import { allHash } from '@shell/utils/promise';
import { CAPI, MANAGEMENT, SNAPSHOT, NORMAN } from '@shell/config/types';
import { MODE, _IMPORT } from '@shell/config/query-params';
import { filterOnlyKubernetesClusters, filterHiddenLocalCluster } from '@shell/utils/cluster';
import { mapFeature, HARVESTER as HARVESTER_FEATURE } from '@shell/store/features';
import { NAME as EXPLORER } from '@shell/config/product/explorer';
import ResourceFetch from '@shell/mixins/resource-fetch';
import { BadgeState } from '@components/BadgeState';

export default {
  components: {
    Banner, ResourceTable, Masthead, BadgeState
  },
  mixins: [ResourceFetch],
  props:  {
    loadIndeterminate: {
      type:    Boolean,
      default: false
    },

    incrementalLoadingIndicator: {
      type:    Boolean,
      default: false
    },

    useQueryParamsForSimpleFiltering: {
      type:    Boolean,
      default: false
    }
  },

  async fetch() {
    this.$initializeFetchData(CAPI.RANCHER_CLUSTER);
    const hash = {
      rancherClusters: this.$fetchType(CAPI.RANCHER_CLUSTER),
      normanClusters:  this.$fetchType(NORMAN.CLUSTER, [], 'rancher'),
      mgmtClusters:    this.$fetchType(MANAGEMENT.CLUSTER),
    };

    if ( this.$store.getters['management/canList'](SNAPSHOT) ) {
      hash.etcdSnapshots = this.$fetchType(SNAPSHOT);
    }

    if ( this.$store.getters['management/canList'](CAPI.MACHINE) ) {
      hash.capiMachines = this.$fetchType(CAPI.MACHINE);
    }

    if ( this.$store.getters['management/canList'](MANAGEMENT.NODE) ) {
      hash.mgmtNodes = this.$fetchType(MANAGEMENT.NODE);
    }

    if ( this.$store.getters['management/canList'](MANAGEMENT.NODE_POOL) ) {
      hash.mgmtPools = this.$fetchType(MANAGEMENT.NODE_POOL);
    }

    if ( this.$store.getters['management/canList'](MANAGEMENT.NODE_TEMPLATE) ) {
      hash.mgmtTemplates = this.$fetchType(MANAGEMENT.NODE_TEMPLATE);
    }

    if ( this.$store.getters['management/canList'](CAPI.MACHINE_DEPLOYMENT) ) {
      hash.machineDeployments = this.$fetchType(CAPI.MACHINE_DEPLOYMENT);
    }

    // Fetch RKE template revisions so we can show when an updated template is available
    // This request does not need to be blocking
    if ( this.$store.getters['management/canList'](MANAGEMENT.RKE_TEMPLATE_REVISION) ) {
      this.$fetchType(MANAGEMENT.RKE_TEMPLATE_REVISION);
    }

    const res = await allHash(hash);

    this.mgmtClusters = res.mgmtClusters;
  },

  data() {
    return {
      resource:     CAPI.RANCHER_CLUSTER,
      schema:       this.$store.getters['management/schemaFor'](CAPI.RANCHER_CLUSTER),
      mgmtClusters: [],
    };
  },

  computed: {
    filteredRows() {
      // If Harvester feature is enabled, hide Harvester Clusters
      if (this.harvesterEnabled) {
        return filterHiddenLocalCluster(filterOnlyKubernetesClusters(this.rows, this.$store), this.$store);
      }

      // Otherwise, show Harvester clusters - these will be shown with a warning
      return filterHiddenLocalCluster(this.rows, this.$store);
    },

    hiddenHarvesterCount() {
      const product = this.$store.getters['currentProduct'];
      const isExplorer = product?.name === EXPLORER;

      // Don't show Harvester banner message on the cluster management page or if Harvester if not enabled
      if (!isExplorer || !this.harvesterEnabled) {
        return 0;
      }

      return this.rows.length - filterOnlyKubernetesClusters(this.rows, this.$store).length;
    },

    createLocation() {
      return {
        name:   'c-cluster-product-resource-create',
        params: {
          product:  this.$store.getters['currentProduct'].name,
          resource: this.resource
        },
      };
    },

    importLocation() {
      return {
        name:   'c-cluster-product-resource-create',
        params: {
          product:  this.$store.getters['currentProduct'].name,
          resource: this.resource
        },
        query: { [MODE]: _IMPORT }
      };
    },

    canImport() {
      const schema = this.$store.getters['management/schemaFor'](CAPI.RANCHER_CLUSTER);

      return !!schema?.collectionMethods.find((x) => x.toLowerCase() === 'post');
    },

    harvesterEnabled: mapFeature(HARVESTER_FEATURE),
  },

  $loadingResources() {
    // results are filtered so we wouldn't get the correct count on indicator...
    return { loadIndeterminate: true };
  },

  mounted() {
    window.c = this;
  },
};
</script>

<template>
  <div>
    <Banner
      v-if="hiddenHarvesterCount"
      color="info"
      :label="t('cluster.harvester.clusterWarning', {count: hiddenHarvesterCount} )"
    />

    <Masthead
      :schema="schema"
      :resource="resource"
      :create-location="createLocation"
      component-testid="cluster-manager-list"
      :show-incremental-loading-indicator="incrementalLoadingIndicator"
      :load-resources="loadResources"
      :load-indeterminate="loadIndeterminate"
    >
      <template
        v-if="canImport"
        slot="extraActions"
      >
        <n-link
          :to="importLocation"
          class="btn role-primary mr-10"
          data-testid="cluster-manager-list-import"
        >
          {{ t('cluster.importAction') }}
        </n-link>
      </template>
    </Masthead>

    <ResourceTable
      :schema="schema"
      :rows="filteredRows"
      :namespaced="false"
      :loading="loading"
      :use-query-params-for-simple-filtering="useQueryParamsForSimpleFiltering"
      :data-testid="'cluster-list'"
      :force-update-live-and-delayed="forceUpdateLiveAndDelayed"
    >
      <!-- Why are state column and subrow overwritten here? -->
      <!-- for rke1 clusters, where they try to use the mgmt cluster stateObj instead of prov cluster stateObj,  -->
      <!-- updates were getting lost. This isn't performant as normal columns, but the list shouldn't grow -->
      <!-- big enough for the performance to matter -->
      <template #cell:state="{row}">
        <BadgeState
          :color="row.stateBackground"
          :label="row.stateDisplay"
        />
      </template>
      <template #sub-row="{fullColspan, row, keyField, componentTestid, i, onRowMouseEnter, onRowMouseLeave}">
        <tr
          v-if="row.stateDescription"
          :key="row[keyField] + '-description'"
          :data-testid="componentTestid + '-' + i + '-row-description'"
          class="state-description sub-row"
          @mouseenter="onRowMouseEnter"
          @mouseleave="onRowMouseLeave"
        >
          <td>&nbsp;</td>
          <td
            :colspan="fullColspan - 1"
            :class="{ 'text-error' : row.stateObj.error }"
          >
            {{ row.stateDescription }}
          </td>
        </tr>
      </template>
      <template #cell:summary="{row}">
        <span v-if="!row.stateParts.length">{{ row.nodes.length }}</span>
      </template>
      <template #cell:explorer="{row}">
        <n-link
          v-if="row.mgmt && row.mgmt.isReady && !row.hasError"
          data-testid="cluster-manager-list-explore-management"
          class="btn btn-sm role-secondary"
          :to="{name: 'c-cluster', params: {cluster: row.mgmt.id}}"
        >
          {{ t('cluster.explore') }}
        </n-link>
        <button
          v-else
          data-testid="cluster-manager-list-explore"
          :disabled="true"
          class="btn btn-sm role-secondary"
        >
          {{ t('cluster.explore') }}
        </button>
      </template>
    </ResourceTable>
  </div>
</template>
