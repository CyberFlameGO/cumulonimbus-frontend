<template>
  <h1>Active Sessions</h1>
  <h2>Who's logged into your account?</h2>
  <div class="quick-action-buttons-container">
    <BackButton fallback="/dashboard" />
    <button
      v-if="!selecting"
      @click="selecting = true"
      :disabled="sessions.loading"
    >
      Bulk Delete
    </button>
    <template v-else>
      <button @click="cancelSelection" :disabled="sessions.loading">
        Cancel
      </button>
      <button @click="confirmDeleteModal!.show()" :disabled="sessions.loading">
        Delete Selected
      </button>
    </template>
  </div>

  <Paginator
    v-model="page"
    @page-change="fetchSessions"
    :item-count="sessions.data?.count || 0"
    :disabled="sessions.loading || !online"
  >
    <Online>
      <template v-if="!sessions.loading">
        <template v-if="!sessions.errored">
          <div
            v-if="sessions.data && sessions.data.count > 0"
            class="content-box-container"
          >
            <SelectableContentBox
              v-for="session in sessions.data.items"
              :title="session.name"
              :selecting="selecting"
              :src="infoIcon"
              theme-safe
              :selected="selected.includes(session.id + '')"
              @click="onSessionClick(session)"
            >
              <strong v-if="session.id === user.account?.session.id"
                >This Session!</strong
              >
              <br v-if="session.id === user.account?.session.id" />
              Click me to manage this session.
            </SelectableContentBox>
          </div>
          <div v-else class="no-content-container">
            <h1>This should not ever be seen.</h1>
            <h2>I am horribly broken.</h2>
          </div>
        </template>
        <div class="no-content-container" v-else>
          <h1>Something went wrong.</h1>
          <button @click="fetchSessions">Retry</button>
        </div>
      </template>
      <div v-else class="no-content-container">
        <LoadingBlurb />
      </div>
    </Online>
  </Paginator>
  <ConfirmModal
    ref="confirmDeleteModal"
    title="Are you sure?"
    @submit="onDeleteSessionsChoice"
    :disabled="sessions.loading"
  >
    <p>Are you sure you want to delete these {{ selected.length }} sessions?</p>
    <p>They will have to be logged back in!</p>
  </ConfirmModal>
  <ConfirmModal
    ref="manageSessionModal"
    title="Manage Session"
    @submit="onManageSessionChoice"
    :disabled="sessions.loading"
  >
    <template v-if="!!selectedSession">
      <code v-text="selectedSession!.name" />
      <br />
      <strong v-if="selectedSession!.id === user.account?.session.id">
        This is your current session.
      </strong>
      <p>
        Created At:
        <code>{{ toDateString(new Date(selectedSession.id * 1000)) }}</code>
      </p>
      <p>
        Expires At:
        <code>{{ toDateString(new Date(selectedSession.exp * 1000)) }}</code>
      </p>
      <p>If you delete this session, you will have to log back in.</p>
    </template>
    <LoadingBlurb v-else />
  </ConfirmModal>
  <FullscreenLoadingBlurb ref="fullscreenLoadingBlurb" />
</template>

<script lang="ts" setup>
  // Vue Components
  import BackButton from '@/components/BackButton.vue';
  import ConfirmModal from '@/components/ConfirmModal.vue';
  import FullscreenLoadingBlurb from '@/components/FullscreenLoadingBlurb.vue';
  import LoadingBlurb from '@/components/LoadingBlurb.vue';
  import Online from '@/components/Online.vue';
  import Paginator from '@/components/Paginator.vue';
  import SelectableContentBox from '@/components/SelectableContentBox.vue';

  // In-House Modules
  import Cumulonimbus from 'cumulonimbus-wrapper';
  import infoIcon from '@/assets/images/info.svg';
  import loadWhenOnline from '@/utils/loadWhenOnline';
  import toDateString from '@/utils/toDateString';

  // Store Modules
  import { sessionsStore } from '@/stores/user-space/sessions';
  import { toastStore } from '@/stores/toast';
  import { userStore } from '@/stores/user';

  // External Modules
  import { ref, onMounted } from 'vue';
  import { useOnline } from '@vueuse/core';

  const confirmDeleteModal = ref<typeof ConfirmModal>(),
    fullscreenLoadingBlurb = ref<typeof FullscreenLoadingBlurb>(),
    manageSessionModal = ref<typeof ConfirmModal>(),
    online = useOnline(),
    page = ref(0),
    selected = ref<string[]>([]),
    selectedSession = ref<Cumulonimbus.Data.Session | null>(null),
    selecting = ref(false),
    sessions = sessionsStore(),
    toast = toastStore(),
    user = userStore();

  async function fetchSessions() {
    if (!online.value) {
      toast.connectivityOffline();
      return;
    }
    window.scrollTo(0, 0);
    try {
      await sessions.getSessions(page.value);
    } catch (e) {
      console.error(e);
      toast.clientError();
    }
  }

  async function onSessionClick(session: Cumulonimbus.Data.Session) {
    if (selecting.value) {
      if (selected.value.includes(session.id + '')) {
        selected.value = selected.value.filter((id) => id !== session.id + '');
      } else {
        selected.value.push(session.id + '');
      }
    } else {
      manageSessionModal.value!.show();
      try {
        selectedSession.value = (
          await user.client!.getSession(session.id + '')
        ).result;
      } catch (e) {
        console.error(e);
        toast.clientError();
        await fetchSessions();
        manageSessionModal.value!.hide();
      }
    }
  }

  onMounted(() =>
    loadWhenOnline(
      fetchSessions,
      !sessions.data || sessions.page !== page.value,
    ),
  );

  async function onManageSessionChoice(choice: boolean) {
    if (choice) {
      fullscreenLoadingBlurb.value!.show();
      const status = await sessions.deleteSession(
        selectedSession.value!.id + '',
      );
      if (status) {
        if (selectedSession.value?.id === user.account?.session.id) {
          fullscreenLoadingBlurb.value!.hide();
          await manageSessionModal.value!.hide();
          await user.logout();
        } else {
          selectedSession.value = null;
          toast.show('Session deleted.');
          await fetchSessions();
          fullscreenLoadingBlurb.value!.hide();
          await manageSessionModal.value!.hide();
        }
      } else fullscreenLoadingBlurb.value!.hide();
    } else {
      await manageSessionModal.value!.hide();
    }
  }

  async function onDeleteSessionsChoice(choice: boolean) {
    if (!online.value) {
      toast.connectivityOffline();
      return;
    }
    if (!choice) {
      await confirmDeleteModal.value!.hide();
      selected.value = [];
      selecting.value = false;
      return;
    }
    try {
      const status = await sessions.deleteSessions(selected.value);
      if (status) {
        if (selected.value.includes(user.account?.session.id + '')) {
          fullscreenLoadingBlurb.value!.hide();
          await manageSessionModal.value!.hide();
          await user.logout();
        } else {
          cancelSelection();
          toast.show(`Deleted ${status} sessions.`);
          await fetchSessions();
          fullscreenLoadingBlurb.value!.hide();
          await manageSessionModal.value!.hide();
        }
      } else fullscreenLoadingBlurb.value!.hide();
    } catch (e) {
      console.error(e);
      toast.clientError();
    }
  }

  function cancelSelection() {
    selected.value = [];
    selecting.value = false;
  }
</script>
