<template>
  <div :class="prefix">
    <div :class="prefix + '__header'" @mousedown="(e) => $emit('handleHeader', e)">
      <div class="title">{{ $t('NODES MAP') }}</div>
      <div class="toolbar">
        <Button
          v-if="groups?.length > 0"
          :icon="isExpand ? 'pi pi-angle-double-down' : 'pi pi-angle-double-up'"
          text
          rounded
          severity="secondary"
          @click.stop="expandAll"
          size="small"
          v-tooltip.top="isExpand ? $t('Collapse All') : $t('Expand All')"
        />
        <slot name="icon" />
      </div>
    </div>
    <div :class="prefix + '__content'">
      <div :class="prefix + '__content-searchbox'">
        <IconField>
          <InputText
            v-model="search"
            class="search-box-input"
            :placeholder="$t('Search by Node ID/Name...')"
            variant="outlined"
            style="width:100%"
          />
          <InputIcon v-if="!search" class="pi pi-search" />
          <InputIcon v-else class="pi pi-times" @click="search = ''" />
        </IconField>
      </div>
      <template v-if="search_nodes?.length > 0 && search">
        <ul>
          <li v-for="(item, i) in search_nodes" :key="i">
            <Node
              :node="item"
              @changeMode="changeNodeMode(item)"
              @mousedown="mouseDown(item, 'node')"
              @mouseup="mouseUp"
            />
          </li>
        </ul>
      </template>
      <template v-else-if="groups_nodes?.length > 0 && !search">
        <ul>
          <Tree
            v-for="(item, i) in groups_nodes"
            :key="i"
            :item="item"
            :index="i"
            :showGroupOnly="showGroupOnly"
            @dragstart="dragstart"
            @dragend="dragend"
            @dragover="dragover"
            @contextmenu="handleContextMenu"
            @changeGroupMode="changeGroupMode"
            @changeNodeMode="changeNodeMode"
            @mousedown="mouseDown"
            @mouseup="mouseUp"
          />
        </ul>
      </template>
      <div v-else class="no_result" style="height:100%">
        <NoResultsPlaceholder
          icon="pi pi-sitemap"
          :title="$t('No Nodes', true)"
          :message="search ? $t('No nodes found in the search') : $t('No nodes found in the map', true)"
        />
      </div>
    </div>
    <ContextMenu ref="menuRef" :model="menuItems" :autoZIndex="false" appendTo="self" />
  </div>
</template>

<script setup>
import IconField from 'primevue/iconfield'
import InputIcon from 'primevue/inputicon'
import InputText from 'primevue/inputtext'
import ContextMenu from "primevue/contextmenu";
import Button from "primevue/button";
import vTooltip from 'primevue/tooltip';
import NoResultsPlaceholder from "@/components/common/noResultsPlaceholder.vue";
import Node from "@/components/sidebar/map/node.vue";
import Tree from './tree.vue';
import {app} from "@/composable/comfyAPI";
import {jumpToNodeId} from "@/composable/node.js";
import {toast} from "@/components/toast";
import { $t } from '@/composable/i18n'
import {ref, watch, defineEmits, computed, onMounted} from "vue";
import {NODE_MODE} from "@/constants/index.js";

const prefix = 'comfyui-easyuse-map-nodes'
// const LOG_PREFIX = '[NodesMap]'

import {storeToRefs} from "pinia";
import {useNodesStore} from "@/stores/nodes.js";
import {getSetting} from "@/composable/settings.js";
const store = useNodesStore()
const {groups_nodes, groups, nodes} = storeToRefs(store)

const showGroupOnly = computed(_=> getSetting('EasyUse.NodesMap.DisplayGroupOnly'))

// ─── Lifecycle ───────────────────────────────────────────────────────────────
onMounted(() => {
  console.log(`[NodesMap420] Component mounted`, {
    groupCount: groups.value?.length ?? 0,
    nodeCount: nodes.value?.length ?? 0,
    showGroupOnly: showGroupOnly.value,
  })
})

// ─── Watchers ────────────────────────────────────────────────────────────────
watch(() => groups_nodes.value?.length, (newVal) => {
  console.log(`${LOG_PREFIX} groups_nodes count changed`, { groupCount: newVal })
})
watch(() => nodes.value?.length, (newVal) => {
  console.log(`${LOG_PREFIX} nodes count changed`, { nodeCount: newVal })
})

// ─── Expand / Collapse ───────────────────────────────────────────────────────
const isExpand = ref(false)
const expandAll = _=>{
  isExpand.value = !isExpand.value
  console.log(`[NodesMap420] ${isExpand.value ? 'Expanding' : 'Collapsing'} all groups`)
  app.canvas.graph._groups.forEach(group=>{
    group.show_nodes = isExpand.value
  })
  store.setGroups(app.canvas.graph._groups)
}

// ─── Search ──────────────────────────────────────────────────────────────────
const search = ref('')
const search_nodes = computed(_=>{
  return nodes?.value.filter(node=> node.id?.toString()?.includes(search.value) || node['type']?.includes(search.value) || node['title']?.includes(search.value)) || []
})

watch(search, (newVal, oldVal) => {
  if (newVal === '') {
    console.log(`[NodesMap420] Search cleared`)
  } else {
    console.log(`[NodesMap420] Search query changed`, {
      query: newVal,
      resultCount: search_nodes.value.length,
    })
  }
})

// ─── Context Menu ─────────────────────────────────────────────────────────────
const menuRef = ref(null)
const targetIsNode = ref(false)
const menuItems = computed(() =>
    targetIsNode.value ?
    [
      {
        label: $t('Jump to this node'),
        icon: 'pi pi-arrow-circle-right',
        visible: true,
        command: () => JumpNode(menuTarget.value),
      },
      {
        label: $t('Rename node'),
        icon: 'pi pi-file-edit',
        visible: true,
        command: () => renameNode(menuTarget.value),
      },
      {
        label: $t('Delete node'),
        icon: 'pi pi-trash',
        command: () => deleteNode(menuTarget.value),
        visible: true,
      },
    ]
    :
    [
      {
        label: $t('Rename group'),
        icon: 'pi pi-file-edit',
        visible: true,
        command: () => renameGroup(menuTarget.value),
      },
      {
        label: $t('Delete group'),
        icon: 'pi pi-trash',
        command: () => deleteGroup(menuTarget.value),
        visible: true,
      },
    ]
)
const menuTarget = ref(null)
const handleContextMenu = (event, groupOrNode) => {
  menuTarget.value = groupOrNode
  targetIsNode.value = groupOrNode.children ? false : true
  console.log(`[NodesMap420] Context menu opened`, {
    targetType: targetIsNode.value ? 'node' : 'group',
    targetId: groupOrNode?.id ?? groupOrNode?.info?.id,
    targetTitle: groupOrNode?.title ?? groupOrNode?.info?.title,
  })
  menuRef.value.show(event)
}

const JumpNode = (node) =>{
  const id = node?.info ? node.info.id : node.id
  console.log(`[NodesMap420] Jumping to node`, { id })
  jumpToNodeId(id)
}

const renameNode = (node) =>{
  const id = node?.info ? node.info.id : node.id
  console.log(`[NodesMap420] Rename node initiated`, { id })
  if(node.info) node.info.is_edit = true
  else node.is_edit = true
}

const deleteNode = (node) =>{
  const id = node.info ? node.info.id : node.id
  console.log(`[NodesMap420] Deleting node`, { id })
  app.canvas.graph.beforeChange()
  app.canvas.graph._nodes.splice(app.canvas.graph._nodes.findIndex(n=>n.id == id),1)
  app.canvas.graph.afterChange()
  app.canvas.setDirty(true, true)
  store.setNodes(app.canvas.graph._nodes)
  console.log(`[NodesMap420] Node deleted`, { id, remainingNodes: app.canvas.graph._nodes.length })
}

const renameGroup = (group) =>{
  const id = group?.info?.id
  console.log(`[NodesMap420] Rename group initiated`, { id })
  group.info.is_edit = true
}

const deleteGroup = (group) =>{
  const id = group?.info?.id
  console.log(`[NodesMap420] Deleting group`, { id })
  app.canvas.graph.beforeChange()
  app.canvas.graph._groups.splice(app.canvas.graph._groups.findIndex(g=>g.id == id),1)
  app.canvas.graph.afterChange()
  app.canvas.setDirty(true, true)
  store.setGroups(app.canvas.graph._groups)
  console.log(`[NodesMap420] Group deleted`, { id, remainingGroups: app.canvas.graph._groups.length })
}

// ─── Timer / Long-press ──────────────────────────────────────────────────────
let pressTimer
let firstTime =0, lastTime =0
let isHolding = false

const changeGroupMode = (item, longpress=false) =>{
  if(isHolding){
    isHolding = false
    return
  }
  const is_always = item.children.find(cate=>cate.mode == NODE_MODE.ALWAYS)
  const newMode = is_always ? (longpress ? NODE_MODE.NEVER : NODE_MODE.BYPASS) : NODE_MODE.ALWAYS
  console.log(`[NodesMap420] Changing group mode`, {
    groupId: item?.info?.id ?? item?.id,
    longpress,
    currentMode: is_always ? 'ALWAYS' : 'other',
    newMode,
  })
  const group_nodes_ids = item.children.map(cate=>cate.id)
  app.canvas.graph._nodes.forEach(node=>{
    if(group_nodes_ids.includes(node.id)){
      node.mode = newMode
      app.graph.change()
    }
  })
  store.setNodes(app.canvas.graph._nodes)
}

const changeNodeMode = (item, longpress=false) =>{
  if(isHolding){
    isHolding = false
    return
  }
  const is_always = item.mode == NODE_MODE.ALWAYS
  const newMode = is_always ? (longpress ? NODE_MODE.NEVER : NODE_MODE.BYPASS) : NODE_MODE.ALWAYS
  console.log(`[NodesMap420] Changing node mode`, {
    nodeId: item?.id,
    longpress,
    currentMode: is_always ? 'ALWAYS' : 'other',
    newMode,
  })
  const node = app.canvas.graph._nodes.find(node=>node.id == item.id)
  if(node){
    node.mode = newMode
    app.graph.change()
    store.setNodes(app.canvas.graph._nodes)
  }
}

const mouseDown = (item, type='group') =>{
  firstTime = new Date().getTime();
  console.log(`[NodesMap420] Mouse down on ${type}`, { id: item?.id ?? item?.info?.id })
  clearTimeout(pressTimer);
  pressTimer = setTimeout(_=>{
    console.log(`[NodesMap420] Long press detected on ${type}`, { id: item?.id ?? item?.info?.id })
    type == 'group' ? changeGroupMode(item,true) : changeNodeMode(item,true)
  },500)
}

const mouseUp = _=>{
  lastTime = new Date().getTime();
  const holdDuration = lastTime - firstTime
  if(holdDuration > 500) {
    isHolding = true
    console.log(`[NodesMap420] Mouse up after long press`, { holdDuration })
  }
  clearTimeout(pressTimer);
}

// ─── Drag & Drop ─────────────────────────────────────────────────────────────
let draggerIndex = ref(null)
let toDragIndex = ref(null)
let isDragging = ref(false)

const dragstart = (e, index) =>{
  draggerIndex.value = index
  console.log(`[NodesMap420] Drag started`, { fromIndex: index, group: app.canvas.graph._groups[index]?.title })
  e.currentTarget.style.opacity = "0.6";
  e.currentTarget.style.border = "1px dashed yellow";
  e.dataTransfer.effectAllowed = 'move';
}

const dragend = (e, index) =>{
  e.target.style.opacity = "1";
  e.currentTarget.style.border = "1px dashed transparent";
  const nodes_map_sorting = getSetting('EasyUse.NodesMap.Sorting')
  console.log(`[NodesMap420] Drag ended`, {
    fromIndex: draggerIndex.value,
    toIndex: toDragIndex.value,
    sortingMode: nodes_map_sorting,
  })
  if(nodes_map_sorting !== 'Manual drag&drop sorting'){
    console.warn(`[NodesMap420] Drag aborted — sorting mode is not 'Manual drag&drop sorting'`, { currentMode: nodes_map_sorting })
    toast.warn($t('For drag and drop sorting, please find Nodes map sorting mode in Settings->EasyUse and change it to manual'))
    return
  }
  let groups = app.canvas.graph._groups
  let dragger_group = groups[draggerIndex.value]
  let todrag_group = groups[toDragIndex.value]
  console.log(`[NodesMap420] Swapping groups`, {
    dragged: dragger_group?.title,
    target: todrag_group?.title,
  })
  app.canvas.graph._groups[draggerIndex.value] = todrag_group
  app.canvas.graph._groups[toDragIndex.value] = dragger_group
  store.setGroups(app.canvas.graph._groups)
}

const dragover = (e, index) =>{
  e.preventDefault();
  if (e.currentIndex == draggerIndex.value) return;
  toDragIndex.value = index
}

defineEmits(['handleHeader'])
</script>
