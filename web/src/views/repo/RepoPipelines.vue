<template>
  <div class="space-y-4">
    <div class="flex">
      <SelectField v-model="selectedEvent" :options="eventOptions" />
    </div>
    <PipelineList :pipelines="displayedPipelines" :loading="isLoading" />
  </div>
</template>

<script lang="ts" setup>
import { computed, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRoute, useRouter } from 'vue-router';

import SelectField from '~/components/form/SelectField.vue';
import PipelineList from '~/components/repo/pipeline/PipelineList.vue';
import useApiClient from '~/compositions/useApiClient';
import { requiredInject } from '~/compositions/useInjectProvide';
import { useWPTitle } from '~/compositions/useWPTitle';
import type { Pipeline } from '~/lib/api/types';
import { WebhookEvents } from '~/lib/api/types/webhook';
import { usePipelineStore } from '~/store/pipelines';

const repo = requiredInject('repo');
const storePipelines = requiredInject('pipelines');
const pipelineStore = usePipelineStore();
const apiClient = useApiClient();
const route = useRoute();
const router = useRouter();
const { t } = useI18n();

const selectedEvent = ref<string>((route.query.event as string) ?? '');
const filteredPipelines = ref<Pipeline[]>([]);
const filterLoading = ref(false);

const isFiltered = computed(() => selectedEvent.value !== '');
const displayedPipelines = computed(() => (isFiltered.value ? filteredPipelines.value : storePipelines.value));
const isLoading = computed(() => (isFiltered.value ? filterLoading.value : pipelineStore.loading));

const eventOptions = [
  { value: '', text: t('repo.pipeline.all_events') },
  ...Object.values(WebhookEvents).map((e) => ({ value: e, text: e })),
];

watch(
  selectedEvent,
  async (event) => {
    const query = { ...route.query };
    if (event) {
      query.event = event;
    } else {
      delete query.event;
    }
    await router.replace({ query });

    if (event) {
      filterLoading.value = true;
      filteredPipelines.value = await apiClient.getPipelineList(repo.value.id, { event });
      filterLoading.value = false;
    }
  },
  { immediate: !!route.query.event },
);

useWPTitle(computed(() => [t('repo.activity'), repo.value.full_name]));
</script>
