<template>
  <div :class="getScrollContainerClass()">
    <SelectGameModalAs></SelectGameModalAs>

    <NGrid class="inventory-summary" cols="1 600:2">
      <NGi class="cell search">
        <NInput placeholder="Filter Items" v-model:value="search" clearable></NInput>
      </NGi>

      <NGi class="cell summary">
        <div class="money" v-html-safe="copperToMoneyString(state.gameState.player.money)"></div>
        <div class="money-brief" v-html-safe="copperToMoneyString(state.gameState.player.money, true)"></div>
        <div class="limit">
          <div class="value bold-cyan">
            <span class="bold-white">{{ getNumItems() }}</span> <span class="black">/</span> {{ getMaxNumItems() }}
          </div>
          <div class="label cyan">items</div>
        </div>
        <div class="limit">
          <div class="value bold-red">
            <span class="bold-white">{{ getWeight() }}</span> <span class="black">/</span> {{ getMaxWeight() }}
          </div>
          <div class="label red">lbs</div>
        </div>
      </NGi>
    </NGrid>

    <NGrid class="inventory" cols="1 800:2 1200:3">
      <NGi v-if="getItems().length == 0">You don't have anything in your inventory.</NGi>

      <NGi v-for="item in getItems()" :key="item.iid">
        <div class="item">
          <div class="name selectable" v-html-safe="ansiToHtml(item.fullName)" :class="getItemNameClass(item)" @click="selectItem(item)"></div>

          <ItemDetails :item="item" :actions="getActions(item)" v-if="selectedIid == item.iid"></ItemDetails>

        </div>
      </NGi>

    </NGrid>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch, defineProps, toRefs } from 'vue'
import { NGrid, NGi, NInput } from 'naive-ui'

import ItemDetails from '@/components/game-modal/ItemDetails.vue'
import SelectGameModalAs from '@/components/game-modal/SelectGameModalAs.vue'

import { state } from '@/static/state'

import { useHelpers } from '@/composables/helpers'
import { useWebSocket } from '@/composables/web_socket'

const { ansiToHtml, copperToMoneyString, getActions } = useHelpers()
const { fetchItems } = useWebSocket()

const props = defineProps(['miniOutputEnabled'])
const { miniOutputEnabled } = toRefs(props)

const items = ref([])
const selectedIid = ref({})
const search = ref('')

function getItems () {
  if (search.value) {
    return items.value.filter(item => item.fullName.toLowerCase().includes(search.value.toLowerCase()))
  } else {
    return items.value
  }
}

function getItemNameClass (item) {
  return selectedIid.value == item.iid ? 'selected' : ''
}

function selectItem (item) {
  if (selectedIid.value == item.iid) {
    selectedIid.value = {}
    return
  }
  selectedIid.value = item.iid
}

function getNumItems() {
  return state.gameState.player.numItems || 0
}

function getMaxNumItems() {
  return state.gameState.player.maxNumItems || 0
}

function getWeight() {
  const initialValue = state.gameState.player.weight || 0
  return Number.isInteger(initialValue) ? initialValue : initialValue.toFixed(2)
}

function getMaxWeight() {
  const initialValue = state.gameState.player.maxWeight || 0
  return Number.isInteger(initialValue) ? initialValue : initialValue.toFixed(2)
}

function getScrollContainerClass () {
  return {
    'scroll-container': true,
    'mini-output-enabled': miniOutputEnabled.value
  }
}

function getInventory () {
  if (state.gameModalAs && state.gameState.charmies[state.gameModalAs]) {
    return state.gameState.charmies[state.gameModalAs].items || []
  }
  return state.gameState.inventory || []
}

let charmieInventoryWatcher = null
function unwatchCharmieInventory () {
  if (charmieInventoryWatcher) {
    charmieInventoryWatcher()
  }
}

function watchCharmieInventory () {
  if (!state.gameModalAs || !state.gameState.charmies[state.gameModalAs]) {
    return
  }

  charmieInventoryWatcher = watch(() => {
    return state.gameState.charmies[state.gameModalAs] ? state.gameState.charmies[state.gameModalAs].items : []
  }, async () => {
    items.value = await fetchItems(getInventory())
  })
}

let watchers = []
onMounted(async () => {
  items.value = await fetchItems(getInventory())
  
  watchers.push(
    watch(() => state.gameState.inventory, async () => {
      if (state.gameModalAs && state.gameState.charmies[state.gameModalAs]) {
        return
      }
      items.value = await fetchItems(state.gameState.inventory)
    })
  )

  watchers.push(
    watch(() => state.gameModalAs, async () => {
      items.value = await fetchItems(getInventory())
      unwatchCharmieInventory()
      watchCharmieInventory()
    })
  )
})

onBeforeUnmount(() => {
  watchers.forEach(w => w())
  unwatchCharmieInventory()
})
</script>

<style lang="less" scoped>
.scroll-container {
  height: calc(100vh - 75px);
  overflow-y: scroll;
  max-width: 1200px;
  margin: 0 auto;

  &.mini-output-enabled {
    height: calc(100vh - 225px);
  }

  .inventory-summary {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    padding: 10px;
    .cell {
      margin: 5px 5px;
    }
    .summary {
      display: flex;
      flex-direction: column;
      justify-content: flex-end;
      .money {
        text-align: right;
      }
      .money-brief {
        display: none;
        text-align: right;
      }
      .limit {
        display: flex;
        flex-direction: row;
        justify-content: flex-end;
        .value {
          text-align: right;
          margin-right: 10px;
        }
        .label {
          text-align: left;
        }
      }
    }
  }
  .inventory {
    .item {
      .name {
        padding: 5px 10px;
        cursor: pointer;
        &.selected {
          background: #121;
        }

        &:hover, &.selected {
          background: #121;
        }
      }
    }
  }
}

@media screen and (max-width: 800px) {
  .scroll-container {
    .inventory-summary {
      .summary {
        .money {
          display: none;
        }
        .money-brief {
          display: block;
        }
      }
    }
  }
}

@media screen and (max-width: 600px) {
  .scroll-container {
    .inventory-summary {
      .summary {
        flex-direction: row;
        justify-content: space-between;
      }
    }
  }
}
</style>