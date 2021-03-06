<script>
import isEmpty from 'lodash/isEmpty';
import merge from 'lodash/merge';

import { mapGetters } from 'vuex';

import Alerting from '@/chart/monitoring/alerting';
import Checkbox from '@/components/form/Checkbox';
import ClusterSelector from '@/chart/monitoring/ClusterSelector';
import Grafana from '@/chart/monitoring/grafana';
import Prometheus from '@/chart/monitoring/prometheus';
import LabeledInput from '@/components/form/LabeledInput';

import { allHash } from '@/utils/promise';
import { STORAGE_CLASS, PVC, SECRET, WORKLOAD_TYPES } from '@/config/types';

const CATTLE_MONITORING_NAMESPACE = 'cattle-monitoring-system';

export default {
  components: {
    Alerting,
    Checkbox,
    ClusterSelector,
    Grafana,
    LabeledInput,
    Prometheus,
  },

  props: {
    chart: {
      type:    Object,
      default: () => ({})
    },

    mode: {
      type:    String,
      default: 'create',
    },

    value: {
      type:    Object,
      default: () => {
        return {};
      },
    },
  },

  data() {
    return {
      accessModes: [
        {
          id:    'ReadWriteOnce',
          label: 'monitoring.accessModes.once',
        },
        {
          id:    'ReadOnlyMany',
          label: 'monitoring.accessModes.readOnlyMany',
        },
        {
          id:    'ReadWriteMany',
          label: 'monitoring.accessModes.many',
        },
      ],
      disableAggregateRoles: false,
      prometheusResources:   [],
      pvcs:                  [],
      secrets:               [],
      storageClasses:        [],
      targetNamespace:       null,
      v1Installed:           false,
    };
  },

  computed: {
    ...mapGetters(['currentCluster']),
    provider() {
      return this.currentCluster.status.provider.toLowerCase();
    },
    workloads() {
      return Object.values(WORKLOAD_TYPES).flatMap(type => this.$store.getters['cluster/all'](type));
    },
  },

  watch: {
    'value.global.rbac.userRoles.create'(createUserRoles) {
      if (createUserRoles) {
        this.disableAggregateRoles = false;
      } else {
        this.value.global.rbac.userRoles.aggregateToDefaultRoles = false;
        this.disableAggregateRoles = true;
      }
    }
  },

  created() {
    if (this.mode === 'create') {
      const extendedDefaults = {
        global: {
          rbac: {
            userRoles: {
              create:                  true,
              aggregateToDefaultRoles: true,
            },
          },
        },
        prometheus: {
          prometheusSpec: {
            scrapeInterval:     '1m',
            evaluationInterval: '1m',
            retention:          '10d',
            retentionSize:      '50GiB',
            enableAdminAPI:     false,
          },
        },
      };

      merge(this.value, extendedDefaults);
    }

    this.$emit('register-before-hook', this.willSave, 'willSave');
  },

  mounted() {
    this.fetchDeps();
  },

  methods: {
    async fetchDeps() {
      const { $store } = this;

      await Promise.all(Object.values(WORKLOAD_TYPES).map(type => this.$store.dispatch('cluster/findAll', { type })));

      this.workloads.forEach((workload) => {
        if (
          !isEmpty(workload?.spec?.template?.spec?.containers) &&
          (workload.spec.template.spec.containers.find(c => c.image.includes('quay.io/coreos/prometheus-operator') ||
            c.image.includes('rancher/coreos-prometheus-operator'))
          ) &&
          workload?.metadata?.namespace !== CATTLE_MONITORING_NAMESPACE
        ) {
          if (!this.v1Installed) {
            this.v1Installed = true;
          }
        }
      });

      if (this.v1Installed) {
        this.$emit('warn', this.t('monitoring.v1Warning', {}, true));

        return;
      }

      const hash = await allHash({
        namespaces:     $store.getters['namespaces'](),
        pvcs:           $store.dispatch('cluster/findAll', { type: PVC }),
        secrets:        $store.dispatch('cluster/findAll', { type: SECRET }),
        storageClasses: $store.dispatch('cluster/findAll', { type: STORAGE_CLASS }),
      });

      await Promise.all(Object.values(WORKLOAD_TYPES).map(type => this.$store.dispatch('cluster/findAll', { type })));

      this.targetNamespace = hash.namespaces[this.chart.targetNamespace] || false;

      if (!isEmpty(hash.storageClasses)) {
        this.storageClasses = hash.storageClasses;
      }

      if (!isEmpty(hash.pvcs)) {
        this.pvcs = hash.pvcs;
      }

      if (!isEmpty(hash.secrets)) {
        this.secrets = hash.secrets;
      }
    },

    willSave() {
      const { prometheusSpec } = this.value.prometheus;
      const selector = prometheusSpec?.storageSpec?.volumeClaimTemplate?.spec?.selector;

      if (selector && isEmpty(selector.matchExpressions) && isEmpty(selector.matchLabels)) {
        delete this.value.prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.selector;
      }
    },
  },
};
</script>

<template>
  <div class="config-monitoring-container">
    <section class=" bordered-section config-cluster-general">
      <div class="row mb-20">
        <ClusterSelector :value="value" :mode="mode" />
      </div>
      <div class="row">
        <div class="col span-6">
          <Checkbox
            v-model="value.global.rbac.userRoles.create"
            label-key="monitoring.createDefaultRoles.label"
            :tooltip="t('monitoring.createDefaultRoles.tip', {}, true)"
          />
        </div>
        <div class="col span-6">
          <Checkbox
            v-model="value.global.rbac.userRoles.aggregateToDefaultRoles"
            label-key="monitoring.aggregateDefaultRoles.label"
            :tooltip="{content:t('monitoring.aggregateDefaultRoles.tip', {}, true), autoHide: false}"
            :disabled="disableAggregateRoles"
          />
        </div>
      </div>
      <div v-if="provider === 'rke'" class="row mt-20">
        <div class="col span-6">
          <LabeledInput
            v-model="value.rkeEtcd.clients.https.certDir"
            :label="t('monitoring.etcdNodeDirectory.label')"
            :tooltip="t('monitoring.etcdNodeDirectory.tooltip', {}, true)"
            :hover-tooltip="true"
            :mode="mode"
          />
        </div>
      </div>
    </section>
    <section class=" bordered-section config-prometheus-container">
      <Prometheus
        v-model="value"
        v-bind="$attrs"
        :access-modes="accessModes"
        :mode="mode"
        :storage-classes="storageClasses"
        :prometheus-pods="prometheusResources"
        :workloads="workloads"
      />
    </section>
    <section class=" bordered-section config-alerting-container">
      <Alerting
        v-model="value"
        :mode="mode"
        :secrets="secrets"
      />
    </section>
    <section class="config-grafana-container">
      <Grafana
        v-model="value"
        :access-modes="accessModes"
        :mode="mode"
        :pvcs="pvcs"
        :storage-classes="storageClasses"
      />
    </section>
  </div>
</template>
