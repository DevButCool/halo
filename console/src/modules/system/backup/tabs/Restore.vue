<script lang="ts" setup>
import UppyUpload from "@/components/upload/UppyUpload.vue";
import { apiClient } from "@/utils/api-client";
import {
  Dialog,
  Toast,
  VAlert,
  VButton,
  VLoading,
  VTabItem,
  VTabs,
} from "@halo-dev/components";
import { useMutation, useQuery } from "@tanstack/vue-query";
import axios from "axios";
import { computed } from "vue";
import { ref } from "vue";
import { useI18n } from "vue-i18n";

const { t } = useI18n();

const complete = ref(false);
const showUploader = ref(false);
const activeTabId = ref("local");

const onProcessCompleted = () => {
  Dialog.success({
    title: t("core.backup.operations.restore.title"),
    description: t("core.backup.operations.restore.description"),
    confirmText: t("core.common.buttons.confirm"),
    showCancel: false,
    async onConfirm() {
      await handleShutdown();
    },
  });
};

async function handleShutdown() {
  await axios.post(`/actuator/restart`);
  Toast.success(t("core.backup.operations.restart.toast_success"));

  setTimeout(() => {
    complete.value = true;
  }, 1000);
}

// Remote download to restore
const remoteDownloadUrl = ref("");
const { isLoading: downloading, mutate: handleRemoteDownload } = useMutation({
  mutationKey: ["remote-download-restore"],
  mutationFn: async () => {
    return await apiClient.migration.restoreBackup({
      downloadUrl: remoteDownloadUrl.value,
    });
  },
  retry: false,
  onSuccess() {
    onProcessCompleted();
  },
});

useQuery({
  queryKey: ["check-health"],
  queryFn: async () => {
    const { data } = await axios.get("/actuator/health");
    return data;
  },
  onSuccess(data) {
    if (data.status === "UP") {
      window.location.reload();
    }
  },
  retry: true,
  retryDelay: 2000,
  enabled: computed(() => complete.value),
});
</script>

<template>
  <div v-if="!complete">
    <div class="px-4 py-3">
      <VAlert :title="$t('core.common.text.tip')" :closable="false">
        <template #description>
          <ul>
            <li>{{ $t("core.backup.restore.tips.first") }}</li>
            <li>
              {{ $t("core.backup.restore.tips.second") }}
            </li>
            <li>
              {{ $t("core.backup.restore.tips.third") }}
            </li>
          </ul>
        </template>
        <template v-if="!showUploader" #actions>
          <VButton @click="showUploader = true">
            {{ $t("core.backup.restore.start") }}
          </VButton>
        </template>
      </VAlert>
    </div>
    <div v-if="showUploader" class="flex flex-col px-4 pb-3">
      <VTabs v-model:active-id="activeTabId" type="pills">
        <VTabItem
          id="local"
          :label="$t('core.backup.restore.tabs.local.label')"
        >
          <UppyUpload
            :restrictions="{
              maxNumberOfFiles: 1,
              allowedFileTypes: ['.zip'],
            }"
            endpoint="/apis/api.console.migration.halo.run/v1alpha1/restorations"
            width="100%"
            @uploaded="onProcessCompleted"
          />
        </VTabItem>
        <VTabItem
          id="remote"
          :label="$t('core.backup.restore.tabs.remote.label')"
        >
          <FormKit
            id="restore-remote-download-form"
            name="restore-remote-download-form"
            type="form"
            :preserve="true"
            @submit="handleRemoteDownload()"
          >
            <FormKit
              v-model="remoteDownloadUrl"
              :label="$t('core.backup.restore.tabs.remote.fields.url')"
              type="text"
              validation="required"
            ></FormKit>
          </FormKit>

          <div class="pt-5">
            <VButton
              :loading="downloading"
              type="secondary"
              @click="$formkit.submit('restore-remote-download-form')"
            >
              {{ $t("core.backup.operations.remote_download.button") }}
            </VButton>
          </div>
        </VTabItem>
      </VTabs>
    </div>
  </div>

  <div v-else class="flex h-72 flex-col items-center justify-center">
    <VLoading />
    <div class="text-xs text-gray-600">
      {{ $t("core.backup.restore.tips.complete") }}
    </div>
  </div>
</template>
