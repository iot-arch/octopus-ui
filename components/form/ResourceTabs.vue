<script>
/*
    Tab component for resource CRU pages featuring:
    Labels and Annotation tabs with content filtered by create-edit-view mixin
    Slots for more tabs, 'before' and 'after' labels and annotations
*/
import Tabbed from '@/components/Tabbed';
import Tab from '@/components/Tabbed/Tab';
import CreateEditView from '@/mixins/create-edit-view';
import KeyValue from '@/components/form/KeyValue';

export default {
  components: {
    Tabbed,
    Tab,
    KeyValue
  },

  mixins: [CreateEditView],

  props: {
    // resource instance
    value: {
      type:    Object,
      default: () => {
        return {};
      }
    },
    // create-edit-view mode
    mode: {
      type:    String,
      default: 'create'
    }
  }
};
</script>

<template>
  <Tabbed v-bind="$attrs" class="mt-20 p-20 card-box-shadow">
    <slot name="before" />
    <Tab name="Labels" :label="t('common.labels')">
      <KeyValue
        key="labels"
        v-model="labels"
        :mode="mode"
        :initial-empty-row="true"
        :pad-left="false"
        :read-allowed="false"
        :protip="false"
        class="pl-10"
      />
    </Tab>
    <Tab name="annotations" :label="t('common.annotations')">
      <KeyValue
        key="annotations"
        v-model="annotations"
        :mode="mode"
        :initial-empty-row="true"
        :pad-left="false"
        :read-allowed="false"
        :protip="false"
        class="pl-10"
      />
    </Tab>
    <slot name="after" />
  </Tabbed>
</template>
