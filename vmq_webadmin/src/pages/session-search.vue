<script setup lang="ts">
import type { DataTableHeader } from '@/plugins/vuetify'
import { Notify } from "@/stores/notification"
import { doVerneMQAPICall, transformTableToArray } from "@/verne-api-helper"

definePage({
  meta: {
    icon: 'mdi-account-search',
    title: 'Session Search',
    drawerIndex: 5,
  },
})

const ALL_NODES = '__all__'

const nodes = ref<any[]>([])
const selectedNode = ref<string>(ALL_NODES)
const searchType = ref<'client_id' | 'user'>('client_id')
const searchValue = ref<string>('')
const searchResults = ref<any[]>([])
const isSearching = ref(false)
const hasSearched = ref(false)
const expanded = ref<any[]>([])

const nodeItems = computed(() => [
  { Node: ALL_NODES, Running: true },
  ...nodes.value,
])

const nodeItemProps = (item: any) => ({
  title: item.Node === ALL_NODES ? 'All Nodes' : item.Node,
  subtitle: item.Node === ALL_NODES ? 'Search all nodes' : (item.Running ? 'Running' : 'Stopped'),
})

const sessionHeaders: DataTableHeader[] = [
  { title: 'Client Id', key: 'client_id' },
  { title: 'User', key: 'user' },
  { title: 'Online', key: 'is_online' },
  { title: 'Offline Msg', key: 'offline_messages' },
  { title: 'Online Msg', key: 'online_messages' },
  { title: 'Peer Host', key: 'peer_host' },
  { title: 'Peer Port', key: 'peer_port' },
  { title: 'Actions', key: 'actions', sortable: false },
]

const reauthorizeSession = async (item: any) => {
  try {
    Notify.info("Reauthorize session")
    await doVerneMQAPICall(item.Node, '/api/v1/session/reauthorize?username=' + item.user + '&client-id=' + item.client_id + '&--mountpoint=' + (item.mountpoint ?? ''))
  } catch (error) {
    Notify.error("Error reauthorizing session " + error)
  }
}

const disconnectSession = async (item: any) => {
  try {
    Notify.info("Disconnect session")
    let path = '/api/v1/session/disconnect?client-id=' + item.client_id
    if (item.mountpoint && item.mountpoint.length > 0)
      path = path + '&--mountpoint=' + item.mountpoint
    await doVerneMQAPICall(item.Node, path)
  } catch (error) {
    Notify.error("Error Disconnect session " + error)
  }
}

const buildSearchPath = (value: string) => {
  // The session/show endpoint only accepts --flag style params (KeySpecs = []).
  // --field=value acts as both column selector and exact-match filter.
  // A leading ~ makes it a regex match: --field=~pattern
  const isClientId = searchType.value === 'client_id'
  const filterParam = isClientId ? `--client_id=${value}` : `--user=${value}`
  const otherField = isClientId ? '--user' : '--client_id'
  return `/api/v1/session/show?${filterParam}&${otherField}&--is_online&--offline_messages&--online_messages&--mountpoint&--peer_host&--peer_port&--queue_started_at&--session_started_at`
}

const search = async () => {
  const trimmed = searchValue.value.trim()
  if (!trimmed) {
    Notify.error("Please enter a search value")
    return
  }

  isSearching.value = true
  hasSearched.value = true

  try {
    const path = buildSearchPath(trimmed)

    if (selectedNode.value === ALL_NODES) {
      const results = await Promise.all(
        nodes.value.map(async (node: any) => {
          const response = await doVerneMQAPICall(node.Node, path)
          return transformTableToArray(response.data, node.Node)
        })
      )
      searchResults.value = results.flat()
    } else {
      const response = await doVerneMQAPICall(selectedNode.value, path)
      searchResults.value = transformTableToArray(response.data, selectedNode.value)
    }
  } catch (error) {
    Notify.error("Error searching sessions: " + error)
    searchResults.value = []
  } finally {
    isSearching.value = false
  }
}

const loadNodes = async () => {
  try {
    const response = await doVerneMQAPICall('self', '/api/v1/cluster/show')
    nodes.value = transformTableToArray(response.data, null)
  } catch (error) {
    Notify.error("Error loading cluster nodes: " + error)
  }
}

onMounted(() => {
  loadNodes()
})
</script>

<template>
  <v-container fluid>
    <v-row>
      <v-col>
        <v-card title="Session Search" subtitle="Search for sessions by Client ID or Username">
          <v-card-text>
            <v-row align="center">
              <v-col cols="12" sm="3">
                <v-select
                  v-model="selectedNode"
                  :items="nodeItems"
                  item-value="Node"
                  :item-props="nodeItemProps"
                  label="Select a Node"
                  density="compact"
                  hide-details
                />
              </v-col>
              <v-col cols="12" sm="auto">
                <v-btn-toggle v-model="searchType" mandatory density="compact" color="primary">
                  <v-btn value="client_id">Client ID</v-btn>
                  <v-btn value="user">Username</v-btn>
                </v-btn-toggle>
              </v-col>
              <v-col cols="12" sm>
                <v-text-field
                  v-model="searchValue"
                  :label="searchType === 'client_id' ? 'Client ID' : 'Username'"
                  density="compact"
                  hide-details
                  clearable
                  @keyup.enter="search"
                />
              </v-col>
              <v-col cols="12" sm="auto">
                <v-btn
                  color="primary"
                  :loading="isSearching"
                  prepend-icon="mdi-magnify"
                  @click="search"
                >
                  Search
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>

          <template v-if="hasSearched">
            <v-divider />
            <v-card-subtitle class="py-2 px-4">
              Found {{ searchResults.length }} result{{ searchResults.length !== 1 ? 's' : '' }}
            </v-card-subtitle>
            <v-data-table
              v-model:expanded="expanded"
              :headers="sessionHeaders"
              :items="searchResults"
              density="compact"
              item-value="client_id"
              :items-per-page="50"
              show-expand
            >
              <template #expanded-row="{ columns, item }">
                <tr>
                  <td :colspan="columns.length">
                    Queue started at {{ new Date(item.queue_started_at).toISOString() }}
                  </td>
                </tr>
                <tr>
                  <td :colspan="columns.length">
                    Session started at {{ new Date(item.session_started_at).toISOString() }}
                  </td>
                </tr>
              </template>
              <template #item.actions="{ item }">
                <BtnIconWithTooltip tooltip="Reauthorize Sessions" myicon="mdi-arrow-u-left-top" @click="reauthorizeSession(item)" />
                <BtnIconWithTooltip tooltip="Disconnect Session" myicon="mdi-arrow-split-vertical" @click="disconnectSession(item)" />
              </template>
            </v-data-table>
          </template>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>
