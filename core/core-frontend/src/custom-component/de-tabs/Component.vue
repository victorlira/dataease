<template>
  <div
    v-if="state.tabShow"
    style="width: 100%; height: 100%"
    :class="[headClass, `ed-tabs-${curThemes}`]"
    class="custom-tabs-head"
    ref="tabComponentRef"
  >
    <de-custom-tab
      v-model="editableTabsValue"
      @tab-add="addTab"
      :addable="isEditMode"
      :font-color="fontColor"
      :active-color="activeColor"
      :border-color="noBorderColor"
      :border-active-color="borderActiveColor"
    >
      <template :key="tabItem.name" v-for="tabItem in element.propValue">
        <el-tab-pane
          class="el-tab-pane-custom"
          :lazy="isEditMode"
          :label="tabItem.title"
          :name="tabItem.name"
        >
          <template #label>
            <div @mousedown.stop>
              <span :style="titleStyle(tabItem.name)">{{ tabItem.title }}</span>
              <el-dropdown
                v-if="isEditMode"
                :effect="curThemes"
                style="line-height: 4 !important"
                trigger="click"
                @command="handleCommand"
              >
                <span class="el-dropdown-link">
                  <el-icon v-if="isEdit"><ArrowDown /></el-icon>
                </span>
                <template #dropdown>
                  <el-dropdown-menu>
                    <el-dropdown-item :command="beforeHandleCommand('editTitle', tabItem)">
                      编辑标题
                    </el-dropdown-item>
                    <el-dropdown-item :command="beforeHandleCommand('copyCur', tabItem)">
                      复制
                    </el-dropdown-item>
                    <el-dropdown-item
                      v-if="element.propValue.length > 1"
                      :command="beforeHandleCommand('deleteCur', tabItem)"
                    >
                      删除
                    </el-dropdown-item>
                  </el-dropdown-menu>
                </template>
              </el-dropdown>
            </div>
          </template>
        </el-tab-pane>
      </template>
      <div
        style="position: absolute; width: 100%; height: 100%"
        :key="tabItem.name + '-content'"
        v-for="(tabItem, index) in element.propValue"
        :class="{ 'switch-hidden': editableTabsValue !== tabItem.name }"
      >
        <de-canvas
          v-if="isEdit && !mobileInPc"
          :ref="'tabCanvas_' + index"
          :component-data="tabItem.componentData"
          :canvas-style-data="canvasStyleData"
          :canvas-view-info="canvasViewInfo"
          :canvas-id="element.id + '--' + tabItem.name"
          :class="moveActive ? 'canvas-move-in' : ''"
          :canvas-active="editableTabsValue === tabItem.name"
        ></de-canvas>
        <de-preview
          v-else
          :ref="'dashboardPreview'"
          :dv-info="dvInfo"
          :cur-gap="curPreviewGap"
          :component-data="tabItem.componentData"
          :canvas-style-data="{}"
          :canvas-view-info="canvasViewInfo"
          :canvas-id="element.id + '--' + tabItem.name"
          :preview-active="editableTabsValue === tabItem.name"
          :show-position="showPosition"
          :outer-scale="scale"
          :outer-search-count="searchCount"
        ></de-preview>
      </div>
    </de-custom-tab>
    <el-dialog
      title="编辑标题"
      :append-to-body="true"
      v-model="state.dialogVisible"
      width="30%"
      :show-close="false"
      :close-on-click-modal="false"
      center
    >
      <el-input
        v-model="state.textarea"
        maxlength="50"
        :placeholder="$t('dataset.input_content')"
      />
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="state.dialogVisible = false">取消</el-button>
          <el-button :disabled="!titleValid" type="primary" @click="sureCurTitle">确认</el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import {
  computed,
  getCurrentInstance,
  nextTick,
  onBeforeMount,
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  toRefs,
  watch
} from 'vue'
import DeCanvas from '@/views/canvas/DeCanvas.vue'
import { dvMainStoreWithOut } from '@/store/modules/data-visualization/dvMain'
import { storeToRefs } from 'pinia'
import { guid } from '@/views/visualized/data/dataset/form/util'
import eventBus from '@/utils/eventBus'
import { canvasChangeAdaptor, findComponentIndexById, isDashboard } from '@/utils/canvasUtils'
import DeCustomTab from '@/custom-component/de-tabs/DeCustomTab.vue'
import DePreview from '@/components/data-visualization/canvas/DePreview.vue'
import { useEmitt } from '@/hooks/web/useEmitt'
import { getPanelAllLinkageInfo } from '@/api/visualization/linkage'
import { dataVTabComponentAdd, groupSizeStyleAdaptor } from '@/utils/style'
import { deepCopyTabItemHelper } from '@/store/modules/data-visualization/copy'
import { snapshotStoreWithOut } from '@/store/modules/data-visualization/snapshot'
const dvMainStore = dvMainStoreWithOut()
const snapshotStore = snapshotStoreWithOut()
const { tabMoveInActiveId, bashMatrixInfo, editMode, mobileInPc } = storeToRefs(dvMainStore)
const tabComponentRef = ref(null)
let carouselTimer = null

const props = defineProps({
  canvasStyleData: {
    type: Object,
    required: true
  },
  canvasViewInfo: {
    type: Object,
    required: true
  },
  dvInfo: {
    type: Object,
    required: true
  },
  element: {
    type: Object,
    default() {
      return {
        propValue: []
      }
    }
  },
  isEdit: {
    type: Boolean,
    default: false
  },
  showPosition: {
    type: String,
    required: false,
    default: 'canvas'
  },
  scale: {
    type: Number,
    required: false,
    default: 1
  },
  // 仪表板刷新计时器
  searchCount: {
    type: Number,
    required: false,
    default: 0
  }
})
const {
  element,
  isEdit,
  showPosition,
  canvasStyleData,
  canvasViewInfo,
  dvInfo,
  scale,
  searchCount
} = toRefs(props)

const state = reactive({
  activeTabName: '',
  curItem: {},
  textarea: '',
  dialogVisible: false,
  tabShow: true
})
const tabsAreaScroll = ref(false)
const editableTabsValue = ref(null)

// 无边框
const noBorderColor = ref('none')
let currentInstance

const isEditMode = computed(() => editMode.value === 'edit' && isEdit.value && !mobileInPc.value)
const curThemes = isDashboard() ? 'light' : 'dark'
const calcTabLength = () => {
  setTimeout(() => {
    if (element.value.propValue.length > 1) {
      const containerDom = document.getElementById(
        'tab-' + element.value.propValue[element.value.propValue.length - 1].name
      )
      tabsAreaScroll.value =
        containerDom.parentNode.clientWidth > tabComponentRef.value.clientWidth - 100
    } else {
      tabsAreaScroll.value = false
    }
  })
}

const beforeHandleCommand = (item, param) => {
  return {
    command: item,
    param: param
  }
}
const curPreviewGap = computed(() =>
  dvInfo.value.type === 'dashboard' && canvasStyleData.value['dashboard'].gap === 'yes'
    ? canvasStyleData.value['dashboard'].gapSize
    : 0
)

function sureCurTitle() {
  state.curItem.title = state.textarea
  state.dialogVisible = false
  snapshotStore.recordSnapshotCache()
}

function addTab() {
  const newName = guid()
  const newTab = {
    name: newName,
    title: '新建Tab',
    componentData: [],
    closable: true
  }
  element.value.propValue.push(newTab)
  editableTabsValue.value = newTab.name
  snapshotStore.recordSnapshotCache()
}

function deleteCur(param) {
  state.curItem = param
  let len = element.value.propValue.length
  while (len--) {
    if (element.value.propValue[len].name === param.name) {
      element.value.propValue.splice(len, 1)
      const activeIndex =
        (len - 1 + element.value.propValue.length) % element.value.propValue.length
      editableTabsValue.value = element.value.propValue[activeIndex].name
      state.tabShow = false
      nextTick(() => {
        state.tabShow = true
      })
    }
  }
}
function copyCur(param) {
  addTab()
  const newTabItem = element.value.propValue[element.value.propValue.length - 1]
  const idMap = {}
  const newCanvasId = element.value.id + '--' + newTabItem.name
  newTabItem.componentData = deepCopyTabItemHelper(newCanvasId, param.componentData, idMap)
  dvMainStore.updateCopyCanvasView(idMap)
}

function editCurTitle(param) {
  state.activeTabName = param.name
  state.curItem = param
  state.textarea = param.title
  state.dialogVisible = true
}

function handleCommand(command) {
  switch (command.command) {
    case 'editTitle':
      editCurTitle(command.param)
      break
    case 'deleteCur':
      deleteCur(command.param)
      snapshotStore.recordSnapshotCache()
      break
    case 'copyCur':
      copyCur(command.param)
      snapshotStore.recordSnapshotCache()
      break
  }
}

const reloadLinkage = () => {
  // 刷新联动信息
  if (dvInfo.value.id) {
    getPanelAllLinkageInfo(dvInfo.value.id).then(rsp => {
      dvMainStore.setNowPanelTrackInfo(rsp.data)
    })
  }
}

const componentMoveIn = component => {
  element.value.propValue.forEach((tabItem, index) => {
    if (editableTabsValue.value === tabItem.name) {
      //获取主画布当前组件的index
      const curIndex = findComponentIndexById(component.id)
      if (curIndex > -1) {
        // 从主画布中移除
        if (isDashboard()) {
          eventBus.emit('removeMatrixItem-canvas-main', curIndex)
          dvMainStore.setCurComponent({ component: null, index: null })
          component.canvasId = element.value.id + '--' + tabItem.name
          const refInstance = currentInstance.refs['tabCanvas_' + index][0]
          if (refInstance) {
            const matrixBase = refInstance.getBaseMatrixSize() //矩阵基础大小
            canvasChangeAdaptor(component, matrixBase)
            component.x = 1
            component.y = 200
            component.style.left = 0
            component.style.top = 0
            tabItem.componentData.push(component)
            refInstance.addItemBox(component) //在适当的时候初始化布局组件
            nextTick(() => {
              refInstance.canvasInitImmediately()
            })
          }
        } else {
          // 从主画布删除
          dvMainStore.deleteComponent(curIndex)
          dvMainStore.setCurComponent({ component: null, index: null })
          component.canvasId = element.value.id + '--' + tabItem.name
          dataVTabComponentAdd(component, element.value.style)
          tabItem.componentData.push(component)
        }
      }
    }
  })

  reloadLinkage()
}

const componentMoveOut = component => {
  if (isDashboard()) {
    canvasChangeAdaptor(component, bashMatrixInfo.value, true)
  }
  // 从Tab画布中移除
  eventBus.emit('removeMatrixItemById-' + component.canvasId, component.id)
  dvMainStore.setCurComponent({ component: null, index: null })
  // 主画布中添加
  if (isDashboard()) {
    eventBus.emit('moveOutFromTab-canvas-main', component)
  } else {
    addToMain(component)
  }
  reloadLinkage()
}

const addToMain = component => {
  const { left, top } = element.value.style
  component.style.left = component.style.left + left
  component.style.top = component.style.top + top
  component.canvasId = 'canvas-main'
  dvMainStore.addComponent({
    component,
    index: undefined,
    isFromGroup: true
  })
}

const moveActive = computed(() => {
  return tabMoveInActiveId.value && tabMoveInActiveId.value === element.value.id
})

const headClass = computed(() => {
  if (tabsAreaScroll.value) {
    return 'tab-head-left'
  } else {
    return 'tab-head-' + element.value.style.headHorizontalPosition
  }
})

const titleStyle = itemName => {
  if (editableTabsValue.value === itemName) {
    return {
      fontSize: (element.value.style.activeFontSize || 18) * scale.value + 'px'
    }
  } else {
    return {
      fontSize: (element.value.style.fontSize || 16) * scale.value + 'px'
    }
  }
}

const fontColor = computed(() => {
  if (
    element.value &&
    element.value.style &&
    element.value.style.headFontColor &&
    typeof element.value.style.headFontColor === 'string'
  ) {
    return element.value.style.headFontColor
  } else {
    return 'none'
  }
})

const activeColor = computed(() => {
  if (
    element.value &&
    element.value.style &&
    element.value.style.headFontActiveColor &&
    typeof element.value.style.headFontActiveColor === 'string'
  ) {
    return element.value.style.headFontActiveColor
  } else {
    return 'none'
  }
})

const borderActiveColor = computed(() => {
  if (
    element.value &&
    element.value.style &&
    element.value.style.headBorderActiveColor &&
    typeof element.value.style.headBorderActiveColor === 'string'
  ) {
    return element.value.style.headBorderActiveColor
  } else {
    return 'none'
  }
})

const titleValid = computed(() => {
  return !!state.textarea && !!state.textarea.trim()
})

watch(
  () => element.value,
  () => {
    calcTabLength()
  },
  { deep: true }
)

watch(
  () => editableTabsValue.value,
  () => {
    nextTick(() => {
      useEmitt().emitter.emit('tabCanvasChange-' + activeCanvasId.value)
    })
  },
  { deep: true }
)

const activeCanvasId = computed(() => {
  return element.value.id + '--' + editableTabsValue.value
})

const reShow = () => {
  state.tabShow = false
  nextTick(() => {
    state.tabShow = true
  })
}

watch(
  () => isEditMode.value,
  () => {
    initCarousel()
  }
)

const initCarousel = () => {
  carouselTimer && clearInterval(carouselTimer)
  carouselTimer = null
  if (!isEditMode.value) {
    if (element.value.carousel?.enable) {
      const switchTime = (element.value.carousel.time || 5) * 1000
      let switchCount = 1
      // 轮播定时器
      carouselTimer = setInterval(() => {
        const nowIndex = switchCount % element.value.propValue.length
        switchCount++
        nextTick(() => {
          editableTabsValue.value = element.value.propValue[nowIndex].name
        })
      }, switchTime)
    }
  }
}

onMounted(() => {
  if (element.value.propValue.length > 0) {
    editableTabsValue.value = element.value.propValue[0].name
  }
  calcTabLength()
  if (['canvas', 'canvasDataV', 'edit'].includes(showPosition.value) && !mobileInPc.value) {
    eventBus.on('onTabMoveIn-' + element.value.id, componentMoveIn)
    eventBus.on('onTabMoveOut-' + element.value.id, componentMoveOut)
    eventBus.on('onTabSortChange-' + element.value.id, reShow)
  }

  currentInstance = getCurrentInstance()
  initCarousel()
  nextTick(() => {
    groupSizeStyleAdaptor(element.value)
  })
})
onBeforeUnmount(() => {
  if (['canvas', 'canvasDataV', 'edit'].includes(showPosition.value) && !mobileInPc.value) {
    eventBus.off('onTabMoveIn-' + element.value.id, componentMoveIn)
    eventBus.off('onTabMoveOut-' + element.value.id, componentMoveOut)
    eventBus.off('onTabSortChange-' + element.value.id, reShow)
  }
})
onBeforeMount(() => {
  if (carouselTimer) {
    clearInterval(carouselTimer)
    carouselTimer = null
  }
})
</script>
<style lang="less" scoped>
:deep(.ed-tabs__content) {
  height: calc(100% - 46px) !important;
}
.ed-tabs-dark {
  :deep(.ed-tabs__new-tab) {
    margin-right: 25px;
    color: #fff;
  }
  :deep(.el-dropdown-link) {
    color: #fff;
  }
}
.ed-tabs-light {
  :deep(.ed-tabs__new-tab) {
    margin-right: 25px;
    background-color: #fff;
  }
}
.el-tab-pane-custom {
  width: 100%;
}
.canvas-move-in {
  border: 2px dotted transparent;
  border-color: blueviolet;
}

.tab-head-left :deep(.ed-tabs__nav-scroll) {
  display: flex;
  justify-content: flex-start;
}

.tab-head-right :deep(.ed-tabs__nav-scroll) {
  display: flex;
  justify-content: flex-end;
}

.tab-head-center :deep(.ed-tabs__nav-scroll) {
  display: flex;
  justify-content: center;
}

.switch-hidden {
  opacity: 0;
  z-index: -1;
}
</style>
