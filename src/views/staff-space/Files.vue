<template>
  <h1>All Files</h1>
  <h2 v-if="files.data">
    Check out everything
    {{
      files.selectedUser ? `${files.selectedUser.username} has ` : ''
    }}uploaded.
  </h2>
  <div class="quick-action-buttons-container">
    <BackButton fallback="/staff" />
    <button
      v-if="!selecting"
      @click="selecting = true"
      :disabled="files.loading"
    >
      Bulk Delete
    </button>
    <template v-else>
      <button @click="cancelSelection" :disabled="files.loading">Cancel</button>
      <button @click="displayModal" :disabled="files.loading">
        Delete Selected
      </button>
    </template>
  </div>

  <Paginator
    v-model="page"
    @page-change="fetchFiles"
    :item-count="files.data ? files.data.count : 0"
    :disabled="files.loading || !online"
  >
    <Online>
      <template v-if="!files.loading">
        <template v-if="!files.errored">
          <div
            v-if="files.data && files.data.count > 0"
            class="content-box-container"
          >
            <PreviewContentBox
              v-for="file in files.data.items"
              :file="file"
              :selecting="selecting"
              :selected="selected.includes(file.id)"
              @click="onFileClick(file)"
            />
          </div>
          <div v-else class="no-content-container">
            <h1>There isn't anything here yet...</h1>
            <h2>
              {{
                files.selectedUser
                  ? `${files.selectedUser.username} hasn't uploaded anything yet.`
                  : 'No one has uploaded anything yet.'
              }}
            </h2>
          </div>
        </template>
        <div class="no-content-container" v-else>
          <h1>Something went wrong.</h1>
          <button @click="fetchFiles">Retry</button>
        </div>
      </template>
      <div class="no-content-container" v-else>
        <LoadingBlurb />
      </div>
    </Online>
  </Paginator>
  <ConfirmModal
    ref="confirmModal"
    title="Are you sure?"
    @submit="deleteSelected"
  >
    <p>Are you sure you want to delete these {{ selected.length }} files?</p>
    <p>They will be lost forever! (A long time!)</p>
  </ConfirmModal>
</template>

<script lang="ts" setup>
  // Vue Components
  import BackButton from '@/components/BackButton.vue';
  import ConfirmModal from '@/components/ConfirmModal.vue';
  import LoadingBlurb from '@/components/LoadingBlurb.vue';
  import Online from '@/components/Online.vue';
  import Paginator from '@/components/Paginator.vue';
  import PreviewContentBox from '@/components/PreviewContentBox.vue';

  // In-House Modules
  import Cumulonimbus from 'cumulonimbus-wrapper';
  import backWithFallback from '@/utils/routerBackWithFallback';
  import defaultErrorHandler from '@/utils/defaultErrorHandler';

  // Store Modules
  import { filesStore } from '@/stores/staff-space/files';
  import { toastStore } from '@/stores/toast';
  import { userStore } from '@/stores/user';

  // External Modules
  import { ref, watch, onMounted } from 'vue';
  import { useOnline } from '@vueuse/core';
  import { useRouter } from 'vue-router';
  import loadWhenOnline from '@/utils/loadWhenOnline';

  const online = useOnline(),
    router = useRouter(),
    files = filesStore(),
    user = userStore(),
    toast = toastStore(),
    selected = ref<string[]>([]),
    selecting = ref(false),
    page = ref(0),
    confirmModal = ref<typeof ConfirmModal>();

  onMounted(async () => {
    loadWhenOnline(
      initPage,
      !files.data ||
        files.page !== page.value ||
        files.selectedUser?.id !== router.currentRoute.value.query.user ||
        (!files.selectedUser && router.currentRoute.value.query.user),
    );
  });

  async function initPage() {
    if (router.currentRoute.value.query.user) {
      try {
        files.selectedUser = (
          await user.client!.getUser(
            router.currentRoute.value.query.user as string,
          )
        ).result;
      } catch (e) {
        if (e instanceof Cumulonimbus.ResponseError) {
          const handled = await defaultErrorHandler(e, router);
          if (!handled) {
            switch (e.code) {
              case 'INVALID_USER_ERROR':
                toast.show('This user does not exist.');
                backWithFallback(router, '/staff/users');
            }
          }
        } else {
          console.error(e);
          toast.clientError();
        }
      }
    } else {
      files.selectedUser = null;
    }
    fetchFiles();
  }

  async function fetchFiles() {
    if (!online.value) {
      toast.connectivityOffline();
      return;
    }
    window.scrollTo(0, 0);
    try {
      await files.getFiles(page.value);
    } catch (e) {
      console.error(e);
      toast.clientError();
    }
  }

  function onFileClick(file: Cumulonimbus.Data.File) {
    if (selecting.value) {
      if (selected.value.includes(file.id)) {
        selected.value = selected.value.filter((f) => f !== file.id);
      } else {
        selected.value.push(file.id);
      }
    }
  }

  function displayModal() {
    if (selected.value.length > 0) {
      confirmModal.value!.show();
    } else {
      toast.show('You must select at least one file to delete.');
    }
  }

  async function deleteSelected(choice: boolean) {
    if (!online.value) {
      toast.connectivityOffline();
      return;
    }
    if (!choice) {
      selecting.value = false;
      selected.value = [];
      confirmModal.value!.hide();
    }
    try {
      const status = await files.deleteFiles(selected.value);
      if (status >= 0) {
        selecting.value = false;
        selected.value = [];
        confirmModal.value!.hide();
        fetchFiles();
      }
    } catch (e) {
      console.error(e);
      toast.clientError();
    }
  }

  function cancelSelection() {
    selecting.value = false;
    selected.value = [];
  }
</script>
