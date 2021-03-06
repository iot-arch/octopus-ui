<script>
import cronstrue from 'cronstrue';
import { cleanUp } from '@/utils/object';
import { CONFIG_MAP, SECRET, WORKLOAD_TYPES, NODE } from '@/config/types';
import Tab from '@/components/Tabbed/Tab';
import CreateEditView from '@/mixins/create-edit-view';
import { allHash } from '@/utils/promise';
import NameNsDescription from '@/components/form/NameNsDescription';
import LabeledSelect from '@/components/form/LabeledSelect';
import LabeledInput from '@/components/form/LabeledInput';
import HealthCheck from '@/components/form/HealthCheck';
import Command from '@/components/form/Command';
import Security from '@/components/form/Security';
import Scheduling from '@/components/form/Scheduling';
import Upgrading from '@/edit/workload/Upgrading';
import Networking from '@/components/form/Networking';
import Footer from '@/components/form/Footer';
import Job from '@/edit/workload/Job';
import { defaultAsyncData } from '@/components/ResourceDetail';
import { _EDIT } from '@/config/query-params';
import ResourceTabs from '@/components/form/ResourceTabs';
import WorkloadPorts from '@/components/form/WorkloadPorts';
import ContainerResourceLimit from '@/components/ContainerResourceLimit';

const workloadTypeOptions = [
  { value: WORKLOAD_TYPES.DEPLOYMENT, label: 'Deployment' },
  { value: WORKLOAD_TYPES.DAEMON_SET, label: 'Daemon Set' },
  { value: WORKLOAD_TYPES.STATEFUL_SET, label: 'Stateful Set' },
  { value: WORKLOAD_TYPES.REPLICA_SET, label: 'Replica Set' },
  { value: WORKLOAD_TYPES.JOB, label: 'Job' },
  { value: WORKLOAD_TYPES.CRON_JOB, label: 'Cron Job' },
  { value: WORKLOAD_TYPES.REPLICATION_CONTROLLER, label: 'Replication Controller' }

];

export default {
  name:       'CruWorkload',
  components: {
    NameNsDescription,
    LabeledSelect,
    LabeledInput,
    Tab,
    Scheduling,
    Upgrading,
    Networking,
    Footer,
    Job,
    ResourceTabs,
    HealthCheck,
    Command,
    Security,
    WorkloadPorts,
    ContainerResourceLimit
  },

  mixins: [CreateEditView],

  props:  {
    value: {
      type:     Object,
      required: true,
    },

    mode:     {
      type:    String,
      default: 'create'
    }
  },

  async fetch() {
    const hash = await allHash({
      configMaps: this.$store.dispatch('cluster/findAll', { type: CONFIG_MAP }),
      secrets:    this.$store.dispatch('cluster/findAll', { type: SECRET }),
      nodes:      this.$store.dispatch('cluster/findAll', { type: NODE })
    });

    this.allSecrets = hash.secrets;
    this.allConfigMaps = hash.configMaps;
    this.allNodes = hash.nodes.map((node) => {
      return { id: node.id };
    });
  },

  asyncData(ctx) {
    let resource;

    if ( !ctx.params.id ) {
      resource = WORKLOAD_TYPES.DEPLOYMENT;
    }

    return defaultAsyncData(ctx, resource);
  },

  data() {
    let type = this.value._type || this.value.type || WORKLOAD_TYPES.DEPLOYMENT;

    if (type === 'workload') {
      type = WORKLOAD_TYPES.DEPLOYMENT;
    }

    let spec = this.value.spec;

    if ( !spec ) {
      spec = { replicas: 1 };
      this.value.spec = spec;
    }

    if (!spec.template) {
      spec.template = { spec: { restartPolicy: this.isJob ? 'Never' : 'Always' } };
    }

    return {
      spec,
      type,
      workloadTypeOptions,
      allConfigMaps:          [],
      allSecrets:             [],
      allNodes:               [],
      showTabs:               false,
    };
  },

  computed: {

    isEdit() {
      return this.mode === _EDIT;
    },

    isJob() {
      return this.type === WORKLOAD_TYPES.JOB || this.isCronJob;
    },

    isCronJob() {
      return this.type === WORKLOAD_TYPES.CRON_JOB;
    },

    isDeployMent() {
      return this.type === WORKLOAD_TYPES.DEPLOYMENT;
    },

    isReplicable() {
      return (this.type === WORKLOAD_TYPES.DEPLOYMENT || this.type === WORKLOAD_TYPES.REPLICA_SET || this.type === WORKLOAD_TYPES.REPLICATION_CONTROLLER || this.type === WORKLOAD_TYPES.STATEFUL_SET);
    },

    // if this is a cronjob, grab pod spec from within job template spec
    podTemplateSpec: {
      get() {
        return this.isCronJob ? this.spec.jobTemplate.spec.template.spec : this.spec.template.spec;
      },
      set(neu) {
        if (this.isJob) {
          this.$set(this.spec.jobTemplate.spec.template, 'spec', neu);
        } else {
          this.$set(this.spec.template, 'spec', neu);
        }
      }
    },

    container: {
      get() {
        if (!this.podTemplateSpec.containers) {
          this.$set(this.podTemplateSpec, 'containers', [{ name: this.value?.metadata?.name, imagePullPolicy: 'Always' }]);
        }

        return this.podTemplateSpec.containers[0];
      },

      set(neu) {
        this.$set(this.podTemplateSpec.containers, 0, neu);
      }
    },

    containerImage: {
      get() {
        return this.container.image;
      },
      set(neu) {
        this.container = { ...this.container, image: neu };
      }
    },

    flatResources: {
      get() {
        const { limits = {}, requests = {} } = this.container.resources || {};
        const { cpu:limitsCpu, memory:limitsMemory } = limits;
        const { cpu:requestsCpu, memory:requestsMemory } = requests;

        return {
          limitsCpu, limitsMemory, requestsCpu, requestsMemory
        };
      },
      set(neu) {
        const {
          limitsCpu, limitsMemory, requestsCpu, requestsMemory
        } = neu;

        const out = {
          requests: {
            cpu:    requestsCpu,
            memory: requestsMemory
          },
          limits: {
            cpu:    limitsCpu,
            memory: limitsMemory
          }
        };

        this.$set(this.container, 'resources', cleanUp(out));
      }
    },

    healthCheck: {
      get() {
        const { readinessProbe, livenessProbe, startupProbe } = this.container;

        return {
          readinessProbe, livenessProbe, startupProbe
        };
      },
      set(neu) {
        Object.assign(this.container, neu);
      }
    },

    command: {
      get() {
        const {
          env, envFrom, command, args, workingDir, stdin, stdinOnce, tty
        } = this.container;

        return {
          env, envFrom, command, args, workingDir, stdin, stdinOnce, tty
        };
      },
      set(neu) {
        Object.assign(this.container, neu);
      }
    },

    schema() {
      return this.$store.getters['cluster/schemaFor']( this.type );
    },

    // show cron schedule in human-readable format
    cronLabel() {
      const { schedule } = this.spec;

      if (!this.isCronJob || !schedule) {
        return null;
      }

      try {
        const hint = cronstrue.toString(schedule);

        return hint;
      } catch (e) {
        return 'invalid cron expression';
      }
    },

    workloadSelector() {
      return { 'workload.user.cattle.io/workloadselector': `${ 'deployment' }-${ this.value.metadata.namespace }-${ this.value.metadata.name }` };
    },

    namespacedSecrets() {
      const namespace = this.value?.metadata?.namespace;

      if (namespace) {
        return this.allSecrets.filter(secret => secret.metadata.namespace === namespace);
      } else {
        return this.allSecrets;
      }
    },

    namespacedConfigMaps() {
      const namespace = this.value?.metadata?.namespace;

      if (namespace) {
        return this.allConfigMaps.filter(configMap => configMap.metadata.namespace === namespace);
      } else {
        return this.allConfigMaps;
      }
    },

  },

  watch: {
    type(neu, old) {
      const template = old === WORKLOAD_TYPES.CRON_JOB ? this.spec?.jobTemplate?.spec?.template : this.spec?.template;

      if (!template.spec) {
        template.spec = {};
      }
      const restartPolicy = this.isJob ? 'Never' : 'Always';

      this.$set(template.spec, 'restartPolicy', restartPolicy);

      if (!this.isReplicable) {
        delete this.spec.replicas;
      }

      if (old === WORKLOAD_TYPES.CRON_JOB) {
        this.$set(this.spec, 'template', { ...template });
        delete this.spec.jobTemplate;
        delete this.spec.schedule;
      } else if (neu === WORKLOAD_TYPES.CRON_JOB) {
        this.$set(this.spec, 'jobTemplate', { spec: { template } });
        this.$set(this.spec, 'schedule', '0 * * * *');
        delete this.spec.template;
      }

      this.$set(this.value, 'type', neu);
      delete this.value.apiVersion;
    },
  },

  methods: {
    goBack() {
      this.$router.go(-1);
    },
    toggleTabs() {
      this.showTabs = !this.showTabs;
    },

    changeSpec(neu) {
      this.$set(this, 'spec', neu);
      this.spec = neu;
      this.$set(this.value, 'spec', neu);
    },

    saveWorkload(cb) {
      if (!this.spec.selector && this.type !== WORKLOAD_TYPES.JOB) {
        this.spec.selector = { matchLabels: this.workloadSelector };
      }

      let template;

      if (this.type === WORKLOAD_TYPES.CRON_JOB) {
        template = this.spec.jobTemplate;
      } else {
        template = this.spec.template;
      }

      if (!template.metadata && this.type !== WORKLOAD_TYPES.JOB) {
        template.metadata = { labels: this.workloadSelector };
      }

      // matchExpressions 'values' are formatted incorrectly; fix them before sending to API
      const nodeAffinity = template?.spec?.affinity?.nodeAffinity || {};
      const preferredDuringSchedulingIgnoredDuringExecution = nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution || [];
      const requiredDuringSchedulingIgnoredDuringExecution = nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution || {};

      preferredDuringSchedulingIgnoredDuringExecution.forEach((term) => {
        const matchExpressions = term?.preference?.matchExpressions || [];

        matchExpressions.forEach((expression) => {
          if (expression.values) {
            expression.values = typeof expression.values === 'string' ? [expression.values] : [...expression.values];
          }
        });
      });

      (requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms || []).forEach((term) => {
        const matchExpressions = term.matchExpressions || [];

        matchExpressions.forEach((expression) => {
          if (expression.values) {
            expression.values = typeof expression.values === 'string' ? [expression.values] : [...expression.values];
          }
        });
      });

      delete this.value.kind;

      if (!this.container.name) {
        this.$set(this.container, 'name', this.value.metadata.name);
      }

      this.save(cb);
    },
  },
};
</script>

<template>
  <form>
    <slot :value="value" name="top">
      <NameNsDescription :value="value" :mode="mode" :extra-columns="['type']">
        <template v-slot:type>
          <LabeledSelect v-model="type" label="Type" :disabled="isEdit" :options="workloadTypeOptions" />
        </template>
      </NameNsDescription>

      <div class="row">
        <div class="col span-4">
          <LabeledInput v-model="containerImage" label="Container Image" placeholder="eg nginx:latest" />
        </div>
        <div class="col span-4">
          <LabeledSelect
            v-model="container.imagePullPolicy"
            :label="t('workload.container.imagePullPolicy')"
            :options="['Always', 'IfNotPresent', 'Never']"
            :mode="mode"
          />
        </div>
        <div class="col span-4" />
        <template v-if="isCronJob">
          <div class="col span-4">
            <LabeledInput v-model="spec.schedule" label="cron Schedule" />
            <span class="cron-hint text-small">{{ cronLabel }}</span>
          </div>
        </template>
        <template v-if="isReplicable">
          <div class="col span-4">
            <LabeledInput v-model.number="spec.replicas" label="Replicas" />
          </div>
        </template>
      </div>
    </slot>

    <ResourceTabs v-model="value" :mode="mode">
      <template #before>
        <Tab v-if="isJob" :label="t('workload.tab.jobConfig')" name="job">
          <Job v-model="spec" class="pl-10" :mode="mode" :type="type" />
        </Tab>
        <Tab name="ports" :label="t('workload.tab.ports')">
          <WorkloadPorts v-model="container.ports" class="pl-10" :mode="mode" />
        </Tab>
        <Tab :label="t('workload.tab.command')" name="command">
          <Command v-model="command" class="pl-10" :mode="mode" :secrets="namespacedSecrets" :config-maps="namespacedConfigMaps" />
        </Tab>
        <Tab :label="t('workload.tab.resource')" name="resources">
          <ContainerResourceLimit v-model="flatResources" class="pl-10" :mode="mode" :show-tip="false" />
        </Tab>
        <Tab :label="t('workload.tab.healthCheck')" name="healthCheck">
          <HealthCheck v-model="healthCheck" class="pl-10" :mode="mode" />
        </Tab>
        <Tab :label="t('workload.tab.securityContext')" name="securityContext">
          <Security v-model="container.securityContext" class="pl-10" :mode="mode" />
        </Tab>
        <Tab :label="t('workload.tab.networking')" name="networking">
          <Networking v-model="podTemplateSpec" class="pl-10" :mode="mode" />
        </Tab>
        <Tab :label="t('workload.tab.nodeScheduling')" name="scheduling">
          <Scheduling v-model="podTemplateSpec" class="pl-10" :mode="mode" :nodes="allNodes" :show-pod="false" />
        </Tab>
        <Tab v-if="isDeployMent" :label="t('workload.tab.scalingUpgradePolicy', undefined, true)" name="upgrading">
          <Upgrading v-model="spec" class="pl-10" :mode="mode" @input="changeSpec" />
        </Tab>
      </template>
    </ResourceTabs>

    <Footer v-if="mode!= 'view'" :errors="errors" :mode="mode" @save="saveWorkload" @done="goBack" />
  </form>
</template>

<style>
  .cron-hint{
    color: var(--muted);
    padding: 3px;
  }
</style>
