<script setup>
// --- 1. 导入依赖 ---
import { ref, computed, nextTick, watch, onMounted, onBeforeUnmount } from 'vue'
import draggable from 'vuedraggable'
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'
import {
  faSun,
  faMoon,
  faPlusCircle,
  faEdit,
  faTimes,
  faEyeSlash,
  faSave,
  faPlus,
  faTrash,
  faUpload,    // <-- (v17.15) 新增
  faClipboard  // <-- (v17.15) 新增
} from '@fortawesome/free-solid-svg-icons'
import { ColorPicker } from 'vue3-colorpicker'
import 'vue3-colorpicker/style.css'
// --- 2. 静态配置数据 ---

/**
 * @description 预设服务器标签
 */
const presets = {
  "mod": { tag: "模组", tag_color_with_hash: "#E67E22" },
  "survival": { tag: "生存", tag_color_with_hash: "#2ECC71" },
  "creative": { tag: "创造", tag_color_with_hash: "#F1C40F" },
  "lobby": { tag: "大厅", tag_color_with_hash: "#3498DB" },
  "proxy": { tag: "代理服", tag_color_with_hash: "#8E44AD" }
}

/**
 * @description (v16 重构) serverTypes 和 labels 仅用于显示
 */
const serverTypeLabels = {
  standalone: '独立服务器',
  parent: '父服务器',
  child: '子服务器'
}

// (v16 建议 5) 移除 defaultJsonString

// --- 3. 核心响应式状态 ---

const config = ref({
  footer: "",
  servers: []
})

const serverTree = ref([])

const jsonInput = ref(``)

// --- 4.5. (新增) 夜间模式状态 ---
const isDarkMode = ref(false)

/**
 * @description 将主题应用到 <html> 标签
 */
function applyTheme(isDark) {
  if (isDark) {
    document.documentElement.classList.add('dark-mode')
    isDarkMode.value = true
  } else {
    document.documentElement.classList.remove('dark-mode')
    isDarkMode.value = false
  }
}

/**
 * @description 切换主题并保存到 localStorage
 */
function toggleTheme() {
  const newThemeState = !isDarkMode.value
  applyTheme(newThemeState)
  localStorage.setItem('theme', newThemeState ? 'dark' : 'light')
}

// (我们将在下面的 onMounted 中调用它)

// --- 4. 模态弹窗状态 ---

// --- (A) 通用 Alert/Confirm 弹窗 ---
const isModalVisible = ref(false)
const modalTitle = ref('')
const modalMessage = ref('')
const modalType = ref('alert')
const modalResolve = ref(null)

// --- (B) (v16 重构) 统一的“添加/编辑”服务器弹窗状态 ---
const isServerModalVisible = ref(false) // 控制新弹窗的显示与隐藏
const modalMode = ref('add')            // 'add' 或 'edit'
const currentServerData = ref(null)     // 存储正在添加/编辑的服务器数据 (副本)
const editingServerIp = ref(null)       // 存储原始 IP，用于编辑时的唯一性检验
const alertModalRef = ref(null)         // Alert/Confirm 弹窗容器
const serverModalRef = ref(null)        // 服务器编辑弹窗容器
const isParentSelectOpen = ref(false)
const parentSelectTriggerRef = ref(null) // 触发器按钮
const parentSelectOptionsRef = ref(null) // 选项列表
const isPresetSelectOpen = ref(false)
const presetSelectTriggerRef = ref(null)
const presetSelectOptionsRef = ref(null)
const isColorPickerOpen = ref(false)
const colorPickerTriggerRef = ref(null)
const colorPickerPanelRef = ref(null)
let cancelModalTimeout = null           // 统一的弹窗清理句柄

const modalIds = {
  alertTitle: 'alert-modal-title',
  alertMessage: 'alert-modal-message',
  serverTitle: 'server-modal-title',
  serverBody: 'server-modal-body'
}

const focusableSelectors = 'a[href], button:not([disabled]), textarea:not([disabled]), input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex^="-"])'

function focusFirstWithin(containerRef) {
  nextTick(() => {
    const container = containerRef?.value ?? containerRef
    if (!container) return
    const preferred = container.querySelector('[data-autofocus]')
    const fallback = container.querySelector(focusableSelectors)
    const target = preferred || fallback || container
    target.focus()
  })
}

watch(isModalVisible, (visible) => {
  if (!visible) return
  focusFirstWithin(alertModalRef)
})

watch(isServerModalVisible, (visible) => {
  if (!visible) return
  focusFirstWithin(serverModalRef)
})

function trapFocus(event, containerRef) {
  const container = containerRef?.value ?? containerRef
  if (!container) return
  const focusable = container.querySelectorAll(focusableSelectors)
  if (focusable.length === 0) {
    event.preventDefault()
    container.focus()
    return
  }
  const first = focusable[0]
  const last = focusable[focusable.length - 1]
  const active = document.activeElement
  if (event.shiftKey) {
    if (active === first || active === container) {
      event.preventDefault()
      last.focus()
    }
  } else if (active === last) {
    event.preventDefault()
    first.focus()
  }
}

function handleGlobalKeydown(event) {
  if (event.key === 'Escape') {
    if (isModalVisible.value) { // ✅ 优先检查 z-index 最高的弹窗
      event.preventDefault()
      onModalCancel()
    }
    // --- (v17.7 新增) 其次检查颜色选择器 ---
    else if (isColorPickerOpen.value) {
      event.preventDefault()
      isColorPickerOpen.value = false
      colorPickerTriggerRef.value?.focus()
    }
    // --- 结束 ---
    else if (isPresetSelectOpen.value) { // (v17.2) 其次检查预设下拉框
      event.preventDefault()
      isPresetSelectOpen.value = false
      presetSelectTriggerRef.value?.focus()
    }
    else if (isParentSelectOpen.value) { // (v17) 其次检查父服下拉框
      event.preventDefault()
      isParentSelectOpen.value = false
      parentSelectTriggerRef.value?.focus() // 焦点返回触发器
    }
    else if (isServerModalVisible.value) {
      event.preventDefault()
      closeServerModal()
    }
  } else if (event.key === 'Tab') {
    // --- (v17.7 新增) ---
    if (isColorPickerOpen.value) {
      isColorPickerOpen.value = false
      trapFocus(event, serverModalRef)
    }
    // --- 结束 ---
    else if (isPresetSelectOpen.value) {
      isPresetSelectOpen.value = false
      trapFocus(event, serverModalRef)
    }
    else if (isParentSelectOpen.value) {
      isParentSelectOpen.value = false
      trapFocus(event, serverModalRef)
    }
    else if (isModalVisible.value) {
      trapFocus(event, alertModalRef)
    } else if (isServerModalVisible.value) {
      trapFocus(event, serverModalRef)
    }
  }
}
// 统一的弹窗清理句柄

// --- 5. 计算属性 (Computed Properties) ---

/**
 * @description 计算出所有可以作为“父服务器”的服务器。
 */
const potentialParentServers = computed(() => {
  if (!config.value.servers) return []
  return config.value.servers.filter(s => s.server_type === 'parent' || s.server_type === 'standalone')
})

/**
 * @description 计算出最终用于“导出”的 JSON 字符串。
 */
const outputJson = computed(() => {
  const cleanConfig = JSON.parse(JSON.stringify(config.value))
  if (cleanConfig.servers) {
    cleanConfig.servers.forEach(server => {
      delete server.tag_color_with_hash
      delete server.selectedPreset
    })
  }
  return JSON.stringify(cleanConfig, null, 2)
})

/**
 * @description (v17) 计算当前选中的父服务器对象，用于自定义下拉框显示
 */
const selectedParent = computed(() => {
  if (!currentServerData.value || !currentServerData.value.parent_ip) {
    return null
  }
  return potentialParentServers.value.find(p => p.ip === currentServerData.value.parent_ip)
})

/**
 * @description (v17.2) 计算当前选中的预设对象，用于自定义下拉框显示
 */
const selectedPresetObject = computed(() => {
  if (!currentServerData.value || !currentServerData.value.selectedPreset) {
    return null
  }
  return presets[currentServerData.value.selectedPreset]
})

// --- 6. 方法 (Methods) ---

// --- (A) 通用 Alert/Confirm 弹窗方法 ---

function showAlert(message, title = '提示') {
  modalTitle.value = title
  modalMessage.value = message
  modalType.value = 'alert'
  isModalVisible.value = true
}

function showConfirm(message, title = '请确认') {
  modalTitle.value = title
  modalMessage.value = message
  modalType.value = 'confirm'
  isModalVisible.value = true
  return new Promise((resolve) => {
    modalResolve.value = resolve
  })
}

function onModalConfirm() {
  isModalVisible.value = false
  if (modalResolve.value) {
    modalResolve.value(true)
  }
  modalResolve.value = null
}

function onModalCancel() {
  isModalVisible.value = false
  if (modalResolve.value) {
    modalResolve.value(false)
  }
  modalResolve.value = null
}

// --- (B) (v16 重构) 统一的“添加/编辑”服务器弹窗方法 ---

/**
 * @description 创建一个用于“添加”弹窗的空白服务器对象。
 * @param {object | null} parentServer - 如果是添加子服，传入父服对象
 * @returns {object} 一个新的服务器数据对象。
 */
function createBlankServer(parentServer = null) {
  return {
    ip: "",
    comment: "",
    tag: "",
    tag_color: "333333",
    tag_color_with_hash: "#333333",
    server_type: "standalone", // 默认为 standalone
    parent_ip: parentServer ? parentServer.ip : "", // (v16) 预设 parent_ip
    selectedPreset: "",
    ignore_in_list: false,
  }
}

/**
 * @description (v16) 打开服务器弹窗（添加或编辑模式）
 * @param {object | null} [serverToEdit=null] - 要编辑的服务器对象。如果为 null，则为添加模式。
 * @param {object | null} [parentServer=null] - (仅添加模式) 如果要添加子服，传入父服。
 */
async function openServerModal(serverToEdit = null, parentServer = null) {
  // 1. (竞争条件修复) 清除任何待处理的关闭超时
  if (cancelModalTimeout) {
    clearTimeout(cancelModalTimeout)
    cancelModalTimeout = null
  }

  // 2. 先清空数据 (确保 v-if 触发)
  currentServerData.value = null

  if (serverToEdit) {
    // --- 编辑模式 ---
    modalMode.value = 'edit'
    // 存储原始 IP 用于验证
    editingServerIp.value = serverToEdit.ip
    // 填充数据为 *深拷贝*
    await nextTick() // 等待 v-if=null 生效
    currentServerData.value = JSON.parse(JSON.stringify(serverToEdit))
  } else {
    // --- 添加模式 ---
    modalMode.value = 'add'
    editingServerIp.value = null
    // 填充数据为空白对象
    await nextTick() // 等待 v-if=null 生效
    currentServerData.value = createBlankServer(parentServer)
  }

  // 3. 显示弹窗
  isServerModalVisible.value = true
}

/**
 * @description (v16) 关闭服务器弹窗
 */
function closeServerModal() {
  isServerModalVisible.value = false

  if (cancelModalTimeout) {
    clearTimeout(cancelModalTimeout)
  }

  // 启动超时清理
  cancelModalTimeout = setTimeout(() => {
    currentServerData.value = null
    editingServerIp.value = null
    cancelModalTimeout = null
  }, 300)
}

/**
 * @description (v16) 保存“添加”或“编辑”的服务器
 */
function saveServer() {
  if (!currentServerData.value) return;

  const server = currentServerData.value;
  const newIp = (server.ip || "").trim();

  // 验证1: 检查 IP 是否为空
  if (newIp === '') {
    showAlert('服务器地址 (IP) 不能为空！', '保存失败');
    return;
  }

  // 验证2: 检查 IP 唯一性
  // 仅当 (IP 发生变化) 或 (这是新添加的服务器) 时才检查
  if (newIp !== editingServerIp.value) {
    const isDuplicate = config.value.servers.some(s => s.ip === newIp);
    if (isDuplicate) {
      showAlert(`服务器地址 (IP) "${newIp}" 已存在！\n请使用唯一的 IP 地址。`, '保存失败');
      return;
    }
  }

  server.ip = newIp

  // (v16) 核心逻辑：根据 parent_ip 自动设置 server_type
  if (server.parent_ip) {
    if (modalMode.value === 'edit' && server.children && server.children.length > 0) {
      showAlert(
        `服务器 [${server.ip}] 已经是一个父服务器 (有 ${server.children.length} 个子服)，它不能被设置为另一个服务器的子服务器。`,
        '保存失败'
      );
      return;
    }
    server.server_type = 'child'

    // 自动转换父服
    const parent = config.value.servers.find(s => s.ip === server.parent_ip)
    if (parent && parent.server_type === 'standalone') {
      parent.server_type = 'parent'
    }
  } else {
    // 如果没有 parent_ip，它就是 standalone
    // （注意：如果它有子服，flattenTreeAndSync 会自动将其设为 parent，这里不用管）
    server.server_type = 'standalone'
  }

  // 应用更改
  if (modalMode.value === 'edit') {
    // --- 编辑模式 ---

    // --- (新增) v17 需求 1: 同步更新子服的 parent_ip ---
    const ipChanged = newIp !== editingServerIp.value;
    if (ipChanged) {
      // 遍历整个服务器列表
      config.value.servers.forEach(s => {
        // 如果有子服的 parent_ip 指向旧 IP
        if (s.parent_ip === editingServerIp.value) {
          // 将它更新为新 IP
          s.parent_ip = newIp;
        }
      });
    }

    // 找到原始服务器对象
    const originalServer = config.value.servers.find(s => s.ip === editingServerIp.value);
    if (originalServer) {
      // 将副本数据覆盖回去
      Object.assign(originalServer, server)
    }
  } else {
    // --- 添加模式 ---
    // (v14) 核心简化：只需推入扁平列表
    config.value.servers.push(server)
  }

  closeServerModal() // 关闭弹窗并清理
}

/**
 * @description 切换父服务器下拉框的显示
 */
function toggleParentSelect() {
  isParentSelectOpen.value = !isParentSelectOpen.value
}

/**
 * @description 选择一个新的父服务器
 */
function selectParent(parentIp) {
  if (currentServerData.value) {
    currentServerData.value.parent_ip = parentIp
  }
  isParentSelectOpen.value = false
  // 将焦点返回给触发器按钮
  parentSelectTriggerRef.value?.focus()
}

/**
 * @description (v17) 处理点击外部以关闭自定义下拉框
 */
function handleClickOutsideParentSelect(event) {
  if (
    isParentSelectOpen.value &&
    parentSelectTriggerRef.value && !parentSelectTriggerRef.value.contains(event.target) &&
    parentSelectOptionsRef.value && !parentSelectOptionsRef.value.contains(event.target)
  ) {
    isParentSelectOpen.value = false
  }
}

/**
 * @description 切换快捷预设下拉框的显示
 */
function togglePresetSelect() {
  isPresetSelectOpen.value = !isPresetSelectOpen.value
}

/**
 * @description 选择一个新的预设
 */
function selectPreset(presetKey) {
  if (currentServerData.value) {
    currentServerData.value.selectedPreset = presetKey
    // 关键: 选择后要立即应用 (applyPreset 会处理空 key)
    applyPreset(currentServerData.value)
  }
  isPresetSelectOpen.value = false
  presetSelectTriggerRef.value?.focus()
}

/**
 * @description (v17.2) 处理点击外部以关闭快捷预设下拉框
 */
function handleClickOutsidePresetSelect(event) {
  if (
    isPresetSelectOpen.value &&
    presetSelectTriggerRef.value && !presetSelectTriggerRef.value.contains(event.target) &&
    presetSelectOptionsRef.value && !presetSelectOptionsRef.value.contains(event.target)
  ) {
    isPresetSelectOpen.value = false
  }
}

// 监听下拉框状态，动态添加/移除全局点击监听
watch(isParentSelectOpen, (isOpen) => {
  if (isOpen) {
    // 使用 mousedown 而非 click，以便在点击事件触发前关闭
    document.addEventListener('mousedown', handleClickOutsideParentSelect)
  } else {
    document.removeEventListener('mousedown', handleClickOutsideParentSelect)
  }
})

/**
 * @description (v17.7) 切换颜色选择器
 */
function toggleColorPicker() {
  isColorPickerOpen.value = !isColorPickerOpen.value
}

/**
 * @description (v17.10) 新增：用于关闭颜色模态框
 */
function closeColorPicker() {
  isColorPickerOpen.value = false
  colorPickerTriggerRef.value?.focus() // 焦点返回
}


// (v17.2) 监听预设下拉框状态
watch(isPresetSelectOpen, (isOpen) => {
  if (isOpen) {
    document.addEventListener('mousedown', handleClickOutsidePresetSelect)
  } else {
    document.removeEventListener('mousedown', handleClickOutsidePresetSelect)
  }
})


// --- (C) 辅助函数 (Helpers) ---

function getContrastColor(hexColor) {
  if (!hexColor || hexColor.length < 7) return '#000000';
  const r = parseInt(hexColor.substr(1, 2), 16);
  const g = parseInt(hexColor.substr(3, 2), 16);
  const b = parseInt(hexColor.substr(5, 2), 16);
  const yiq = ((r * 299) + (g * 587) + (b * 114)) / 1000;
  return (yiq >= 128) ? '#000000' : '#FFFFFF';
}

function sanitizeIpForId(ip) {
  if (!ip) return 'new-server';
  return ip.replace(/[^a-zA-Z0-9_-]/g, '_');
}

/**
 * @description (v14) 核心重构：将扁平的服务器列表转换为嵌套树形结构。
 */
function buildTree(flatList) {
  const map = {}
  const serversWithChildren = flatList
    .map(server => {
      const serverCopy = { ...server, children: [] }
      map[serverCopy.ip] = serverCopy
      return serverCopy
    })

  const tree = []

  serversWithChildren.forEach(server => {
    if (server.parent_ip) {
      const parent = map[server.parent_ip]
      if (parent) {
        server.server_type = 'child'
        parent.children.push(server)
      } else {
        // 孤儿节点
        server.parent_ip = ''
        server.server_type = 'standalone'
        tree.push(server)
      }
    } else {
      // 根节点
      tree.push(server)
    }
  })

  return tree
}


// --- (D) 核心逻辑 (Core Logic) ---

/**
 * @description 解析 JSON 字符串并设置到 `config` 状态中。
 */
function parseAndSetConfig(jsonString) {
  try {
    const data = JSON.parse(jsonString)
    if (data.servers && Array.isArray(data.servers)) {
      const ipSet = new Set();
      const duplicates = [];

      data.servers.forEach((s) => {
        if (ipSet.has(s.ip)) {
          duplicates.push(s.ip);
        }
        ipSet.add(s.ip);

        // 补全 UI 辅助属性
        s.tag_color_with_hash = (s.tag_color && s.tag_color.length > 0) ? '#' + s.tag_color : '#888888'
        s.selectedPreset = ""
        s.parent_ip = s.parent_ip || ""
        s.ignore_in_list = s.ignore_in_list || false
        s.comment = s.comment || ""
      })

      if (duplicates.length > 0) {
        throw new Error(`导入失败：JSON 数据中包含重复的 IP 地址。\n重复项: ${[...new Set(duplicates)].join(', ')}`);
      }

      data.servers.sort((a, b) => a.priority - b.priority)
    } else {
      data.servers = []
    }
    config.value = data
  } catch (e) {
    showAlert(e.message, "导入失败")
  }
}

/**
 * @description “加载配置”按钮的点击事件处理器。
 */
function loadConfig() {
  parseAndSetConfig(jsonInput.value)
}

/**
 * @description (v14) 拖拽结束后，重建 `config.value.servers`。
 */
function flattenTreeAndSync() {
  const newFlatList = []
  let priorityCounter = 0

  function traverse(nodes, parentIp = "") {
    if (!nodes) return

    nodes.forEach((server) => {
      server.parent_ip = parentIp

      if (parentIp) {
        server.server_type = 'child'
      } else if (server.children.length > 0) {
        server.server_type = 'parent'
      } else {
        server.server_type = 'standalone'
      }

      server.priority = (priorityCounter + 1) * 10
      priorityCounter++

      const { children, ...flatServer } = server
      newFlatList.push(flatServer)

      traverse(children, server.ip)
    })
  }

  traverse(serverTree.value, "")

  // 临时禁用 watch，更新，然后再启用
  stopWatch()
  config.value.servers = newFlatList
  nextTick(() => {
    startWatch()
  })
}

/**
 * @description (v15) 拖拽规则。
 */
function checkMove(moveEvent) {
  const draggedEl = moveEvent.draggedContext.element
  const toEl = moveEvent.to

  // 检查被拖拽的元素是否是一个 "父服" (有子节点)
  if (
    draggedEl &&
    draggedEl.children &&
    draggedEl.children.length > 0
  ) {
    // 检查目标列表是否是一个 "子列表"
    if (toEl && toEl.classList && toEl.classList.contains('child-list')) {
      // 阻止移动 (父服不能成为子服)
      return false
    }
  }
  return true
}

// --- (E) 弹窗内的表单逻辑 (颜色与预设) ---

function updateColorFromPicker(server) {
  server.tag_color = server.tag_color_with_hash.substring(1).toUpperCase()
}

function applyPreset(server) {
  const presetKey = server.selectedPreset
  if (!presetKey || !presets[presetKey]) return

  const preset = presets[presetKey]
  server.tag = preset.tag
  server.tag_color_with_hash = preset.tag_color_with_hash
  updateColorFromPicker(server)
}

function checkIfCustom(server) {
  if (!server.selectedPreset || !presets[server.selectedPreset]) return
  const preset = presets[server.selectedPreset]

  if (server.tag !== preset.tag || server.tag_color_with_hash !== preset.tag_color_with_hash) {
    server.selectedPreset = ""
  }
}

function onColorInput(server) {
  updateColorFromPicker(server)
  checkIfCustom(server)
}

// --- (F) 服务器增删 (根级别) ---

/**
 * @description “+ 添加服务器”按钮 (v16)
 */
const addServer = () => openServerModal(null, null)

/**
 * @description “+ 子服”按钮 (v16)
 */
const addChildServer = (parent) => openServerModal(null, parent)

/**
 * @description 删除一个服务器（及其所有子服务器）。
 */
async function removeServer(server) {
  const serverToRemove = server

  let confirmed = false
  if (serverToRemove.server_type === 'parent' || serverToRemove.server_type === 'standalone') {
    const childCount = config.value.servers.filter(s => s.parent_ip === serverToRemove.ip).length
    const message = `确定要删除服务器 ${serverToRemove.ip} 吗？${childCount > 0 ? `\n(其 ${childCount} 个子服务器将一并删除)` : ''}`
    confirmed = await showConfirm(message, '删除确认');
  } else {
    confirmed = await showConfirm(`确定要删除服务器 ${serverToRemove.ip} 吗？`, '删除确认');
  }

  if (confirmed) {
    if (serverToRemove.server_type === 'parent' || serverToRemove.server_type === 'standalone') {
      // 删除父服及其所有子服
      const parentIp = serverToRemove.ip
      config.value.servers = config.value.servers.filter(s => {
        return s.ip !== serverToRemove.ip && s.parent_ip !== parentIp
      })
    } else {
      // 只删除一个子服
      const index = config.value.servers.findIndex(s => s.ip === serverToRemove.ip);
      if (index > -1) {
        config.value.servers.splice(index, 1)
      }
    }
  }
}

/**
 * @description (v16 建议 4) 全局删除所有服务器
 */
async function removeAllServers() {
  const confirmed = await showConfirm(
    `您确定要删除所有 ${config.value.servers.length} 个服务器吗？\n此操作不可撤销。`,
    '删除全部确认'
  );
  if (confirmed) {
    config.value.servers = [];
    // (可选) 也可以重置页脚
    // config.value.footer = "";
  }
}

// --- (G) 导入/导出 (IO) 操作 ---

function copyToClipboard() {
  navigator.clipboard.writeText(outputJson.value).then(() => {
    showAlert('已复制到剪贴板！', '复制成功')
  }, () => {
    showAlert('复制失败！', '复制失败')
  })
}

// --- (H) v14 核心重构：双向同步 Watch ---
//
const stopWatch = watch(
  () => config.value.servers,
  (newFlatList) => {
    serverTree.value = buildTree(newFlatList)
  },
  {
    deep: true,
    immediate: true
  }
)

const startWatch = () => {
  stopWatch() // 确保旧的已停止
  watch(
    () => config.value.servers,
    (newFlatList) => {
      serverTree.value = buildTree(newFlatList)
    },
    { deep: true, immediate: true }
  )
}
// --- 7. 初始化 ---

// (v16 建议 5) 组件加载时，尝试从剪贴板自动导入
onMounted(async () => {
  try {
    // 1. 尝试读取剪贴板
    const text = await navigator.clipboard.readText();
    if (!text) return; // 剪贴板为空

    // 2. 尝试解析为 JSON
    const data = JSON.parse(text);

    // 3. 验证是否为我们的配置格式 (必须有 servers 数组 和/或 footer)
    if (data && (Array.isArray(data.servers) || data.hasOwnProperty('footer'))) {
      jsonInput.value = text; // 填充到导入框
      parseAndSetConfig(text); // 自动导入
      // (v16 建议 5) 保持为空，但提示用户
      // showAlert('已从剪贴板自动导入配置。', '导入成功');
    }
  } catch (e) {
    // 失败 (权限被拒、剪贴板不是 JSON、JSON 格式不对等)
    // 在控制台打印错误，但不要打扰用户
    console.warn('Failed to auto-import from clipboard:', e.message);
  }

  // (v16 建议 5) 确保初始列表为空 (而不是依赖 defaultJsonString)
  if (config.value.servers.length === 0) {
    parseAndSetConfig(`{"footer": "", "servers": []}`);
  }
  const savedTheme = localStorage.getItem('theme')
  if (savedTheme) {
    // 1. 优先使用本地存储的设置
    applyTheme(savedTheme === 'dark')
  } else {
    // 2. 否则，跟随操作系统的偏好
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
    applyTheme(prefersDark)
  }
  document.addEventListener('keydown', handleGlobalKeydown);
});

onBeforeUnmount(() => {
  document.removeEventListener('keydown', handleGlobalKeydown);
  document.removeEventListener('mousedown', handleClickOutsideParentSelect);
  document.removeEventListener('mousedown', handleClickOutsidePresetSelect);
});

// (v16 建议 5) 移除末尾的 parseAndSetConfig(defaultJsonString)

</script>

<template>
  <div class="layout-container">

    <div class="panel editor-panel">
      <header class="panel-header">

        <button @click="toggleTheme" class="theme-toggle-btn" :title="isDarkMode ? '切换到日间模式' : '切换到夜间模式'">
          <font-awesome-icon v-if="isDarkMode" :icon="faSun" />
          <font-awesome-icon v-else :icon="faMoon" />
        </button>
        <h1>服务器配置编辑器</h1>
        <div class="subtitle">拖拽服务器卡片调整优先级</div>
      </header>

      <div class="panel-body">

        <div class="form-section">
          <h3>页脚设置</h3>
          <div class="form-group">
            <input type="text" v-model="config.footer" placeholder="输入页脚文本" />
          </div>
        </div>

        <div class="form-section server-list-section">
          <div class="server-list-header">
            <h3>服务器列表</h3>
            <div class="header-actions">
              <button @click="removeAllServers" class="btn btn-danger" v-if="config.servers.length > 0">
                <font-awesome-icon :icon="faTrash" /> 全部删除
              </button>
              <button @click="addServer" class="btn btn-add">
                + 添加服务器
              </button>
            </div>
          </div>

          <draggable v-model="serverTree" :item-key="server => server.ip" handle=".drag-handle"
            :group="{ name: 'servers', pull: true, put: true }" :move="checkMove" @end="flattenTreeAndSync"
            class="server-list" :name="'server-list-anim-root'">
            <template #item="{ element: server }">
              <div :key="server.ip" class="server-item-container" :class="{
                'is-parent-container': server.children.length > 0
              }">
                <div class="server-item-simple" :class="{
                  'is-parent': server.children.length > 0,
                  'is-standalone': server.children.length === 0,
                  'is-ignored': server.ignore_in_list
                }">
                  <div class="drag-handle">⠿</div>
                  <div class="simple-info">
                    <span v-if="server.tag" class="simple-tag" :style="{
                      backgroundColor: server.tag_color_with_hash,
                      color: getContrastColor(server.tag_color_with_hash)
                    }">
                      {{ server.tag }}
                    </span>
                    <span class="simple-comment" v-if="server.comment">{{ server.comment }}</span>
                    <span class="simple-ip" :class="{ 'with-comment': server.comment }">
                      {{ server.comment ? '(' + server.ip + ')' : server.ip }}
                    </span>
                    <span v-if="server.ignore_in_list" class="simple-ignored-badge">(已隐藏)</span>
                  </div>
                  <div class="simple-actions">
                    <button @click="addChildServer(server)" class="btn btn-add-child-simple btn-icon-simple"
                      title="添加子服务器">
                      <font-awesome-icon :icon="faPlus" />
                    </button>
                    <button @click="openServerModal(server)" class="btn btn-edit-simple btn-icon-simple" title="编辑服务器">
                      <font-awesome-icon :icon="faEdit" />
                    </button>
                    <button @click="removeServer(server)" class="btn btn-danger btn-remove-simple btn-icon-simple"
                      title="删除服务器">
                      <font-awesome-icon :icon="faTrash" />
                    </button>
                  </div>
                </div>

                <draggable v-model="server.children" :item-key="child => child.ip" handle=".drag-handle"
                  :group="{ name: 'servers', pull: true, put: true }" @end="flattenTreeAndSync"
                  class="server-list child-list" :name="'server-list-anim-child'">
                  <template #item="{ element: childServer }">
                    <div :key="childServer.ip" class="server-item-simple is-child" :class="{
                      'is-ignored': childServer.ignore_in_list
                    }">
                      <div class="drag-handle">⠿</div>
                      <div class="simple-info">
                        <span v-if="childServer.tag" class="simple-tag" :style="{
                          backgroundColor: childServer.tag_color_with_hash,
                          color: getContrastColor(childServer.tag_color_with_hash)
                        }">
                          {{ childServer.tag }}
                        </span>
                        <span class="simple-comment" v-if="childServer.comment">{{ childServer.comment }}</span>
                        <div class="simple-info-line2">
                          <span class="simple-ip" :class="{ 'with-comment': childServer.comment }">
                            {{ childServer.comment ? '(' + childServer.ip + ')' : childServer.ip }}
                          </span>
                          <span v-if="childServer.ignore_in_list" class="simple-ignored-badge">(已隐藏)</span>
                        </div>
                      </div>
                      <div class="simple-actions">
                        <button @click="openServerModal(childServer)" class="btn btn-edit-simple btn-icon-simple"
                          title="编辑服务器">
                          <font-awesome-icon :icon="faEdit" />
                        </button>
                        <button @click="removeServer(childServer)"
                          class="btn btn-danger btn-remove-simple btn-icon-simple" title="删除服务器">
                          <font-awesome-icon :icon="faTrash" />
                        </button>
                      </div>
                    </div>
                  </template>
                </draggable>
              </div>
            </template>
          </draggable>

          <div v-if="!serverTree || serverTree.length === 0" class="empty-state">
            <h3>尚未添加服务器</h3>
            <p>点击上方"添加"按钮开始配置，或从剪贴板自动导入</p>
          </div>
        </div>
      </div>
    </div>

    <div class="panel io-panel">
      <div class="panel-body">
        <div class="form-section">
          <h3 class="io-header">1. 导入 (Import)</h3>
          <p>从机器人 `/mcs export` 导出的 JSON 粘贴到此处：</p>
          <div class="form-group">
            <textarea v-model="jsonInput" rows="8" placeholder="在此粘贴 JSON..."></textarea>
          </div>
          <button @click="loadConfig" class="btn btn-primary">
            <font-awesome-icon :icon="faUpload" /> 加载配置
          </button>
        </div>

        <div class="form-section">
          <h3 class="io-header">2. 导出 (Export)</h3>
          <p>复制下面的内容，用于机器人 `/mcs import` 命令：</p>
          <div class="form-group">
            <textarea :value="outputJson" rows="15" readonly></textarea>
          </div>
          <button @click="copyToClipboard" class="btn btn-secondary">
            <font-awesome-icon :icon="faClipboard" /> 复制到剪贴板
          </button>
        </div>
      </div>
    </div>

    <transition name="modal-fade">
      <div v-if="isModalVisible" class="modal-overlay" role="presentation" @click.self="onModalCancel">
        <div class="modal-box" ref="alertModalRef" role="dialog" aria-modal="true"
          :aria-labelledby="modalIds.alertTitle" :aria-describedby="modalIds.alertMessage" tabindex="-1">
          <div class="modal-header">
            <h3 :id="modalIds.alertTitle">{{ modalTitle }}</h3>
          </div>
          <div class="modal-body">
            <pre :id="modalIds.alertMessage">{{ modalMessage }}</pre>
          </div>
          <div class="modal-footer">
            <button v-if="modalType === 'confirm'" @click="onModalCancel" class="btn btn-modal-cancel"
              type="button">取消</button>
            <button @click="onModalConfirm" class="btn btn-modal-confirm" type="button">确认</button>
          </div>
        </div>
      </div>
    </transition>


    <transition name="modal-fade">
      <div v-if="isServerModalVisible" class="modal-overlay edit-modal" role="presentation"
        @click.self="closeServerModal">
        <div class="modal-box edit-modal-box" ref="serverModalRef" role="dialog" aria-modal="true"
          :aria-labelledby="modalIds.serverTitle" :aria-describedby="modalIds.serverBody" tabindex="-1">

          <div class="modal-header">
            <h3 v-if="modalMode === 'add'" :id="modalIds.serverTitle">
              <font-awesome-icon :icon="faPlusCircle" /> 添加新服务器
            </h3>
            <h3 v-else :id="modalIds.serverTitle">
              <font-awesome-icon :icon="faEdit" /> 编辑服务器 {{ editingServerIp }}
            </h3>
            <button @click="closeServerModal" class="btn-close-modal" type="button">
              <font-awesome-icon :icon="faTimes" />
            </button>
          </div>

          <div class="modal-body" v-if="currentServerData" :id="modalIds.serverBody">
            <div class="server-form">

              <div class="form-row">
                <div class="form-group grow">
                  <label>服务器地址 (IP) <span class="required">*</span></label>
                  <input type="text" v-model="currentServerData.ip" placeholder="例如: play.example.com" data-autofocus />
                </div>
              </div>

              <div class="form-row">
                <div class="form-group grow">
                  <label>注释 (Comment) (可选)</label>
                  <input type="text" v-model="currentServerData.comment" placeholder="例如: 生存一区 (S1)" />
                </div>
              </div>

              <div class="form-row">

                <div class="form-group" style="flex-grow: 1;"> <label>标签 / 颜色</label>
                  <div class="form-compound-input">
                    <input type="text" v-model="currentServerData.tag" @input="checkIfCustom(currentServerData)"
                      placeholder="留空则不显示" class="form-compound-input-text" />

                    <div class="color-picker-wrapper">
                      <button type="button" class="custom-color-trigger" ref="colorPickerTriggerRef"
                        :style="{ backgroundColor: currentServerData.tag_color_with_hash }"
                        @click.stop="toggleColorPicker">
                      </button>

                      <transition name="modal-fade">
                        <div v-if="isColorPickerOpen" class="color-picker-modal-overlay" @click.self="closeColorPicker"
                          role="presentation">
                          <div class="color-picker-modal-box" ref="colorPickerPanelRef" role="dialog" aria-modal="true">
                            <ColorPicker is-widget format="hex" :disable-alpha="true"
                              v-model:pureColor="currentServerData.tag_color_with_hash"
                              @pureColorChange="onColorInput(currentServerData)" />
                            <button type="button" class="btn btn-modal-confirm btn-color-picker-done"
                              @click="closeColorPicker">
                              完成
                            </button>
                          </div>
                        </div>
                      </transition>
                    </div>
                  </div>
                </div>

                <div class="form-group" style="flex-grow: 1;">
                  <label>快捷预设</label>
                  <div class="custom-select-container">

                    <button type="button" class="custom-select-trigger" ref="presetSelectTriggerRef"
                      @click="togglePresetSelect" aria-haspopup="listbox" :aria-expanded="isPresetSelectOpen">

                      <span v-if="selectedPresetObject" class="selected-option-content">
                        <span class="simple-tag-small" :style="{
                          backgroundColor: selectedPresetObject.tag_color_with_hash,
                          color: getContrastColor(selectedPresetObject.tag_color_with_hash)
                        }">{{ selectedPresetObject.tag }}</span>
                        <span class="option-text">{{ selectedPresetObject.tag }}</span>
                      </span>
                      <span v-else class="option-placeholder">-- 自定义 --</span>

                      <span class="custom-select-arrow" :class="{ 'is-open': isPresetSelectOpen }">▼</span>
                    </button>

                    <transition name="modal-fade">
                      <ul v-if="isPresetSelectOpen" class="custom-select-options" ref="presetSelectOptionsRef"
                        role="listbox">

                        <li class="custom-select-option" :class="{ 'is-selected': !currentServerData.selectedPreset }"
                          @click="selectPreset('')" role="option" :aria-selected="!currentServerData.selectedPreset">
                          <span class="option-placeholder">-- 自定义 --</span>
                        </li>

                        <li v-for="(preset, key) in presets" :key="key" class="custom-select-option" :class="{
                          'is-selected': currentServerData.selectedPreset === key
                        }" @click="selectPreset(key)" role="option"
                          :aria-selected="currentServerData.selectedPreset === key">

                          <span class="simple-tag-small" :style="{
                            backgroundColor: preset.tag_color_with_hash,
                            color: getContrastColor(preset.tag_color_with_hash)
                          }">{{ preset.tag }}</span>

                          <span class="option-text">{{ preset.tag }}</span>
                        </li>
                      </ul>
                    </transition>
                  </div>
                </div>

              </div>
              <p class="form-help-text" style="margin-top: -10px; margin-bottom: 18px;text-align: right;">
                💡 选择预设可自动填充标签和颜色。
              </p>


              <div class="form-row">
                <div class="form-group grow">
                  <label>父服务器 (Parent IP)</label>
                  <div class="custom-select-container">

                    <button type="button" class="custom-select-trigger" ref="parentSelectTriggerRef"
                      @click="toggleParentSelect" :disabled="(modalMode === 'edit' && currentServerData.children && currentServerData.children.length > 0) ||
                        (potentialParentServers.length === 0)
                        " aria-haspopup="listbox" :aria-expanded="isParentSelectOpen">

                      <span v-if="selectedParent" class="selected-option-content">
                        <span v-if="selectedParent.tag" class="simple-tag-small" :style="{
                          backgroundColor: selectedParent.tag_color_with_hash,
                          color: getContrastColor(selectedParent.tag_color_with_hash)
                        }">{{ selectedParent.tag }}</span>
                        <span class="option-text">
                          <template v-if="selectedParent.comment">{{ selectedParent.comment }} ({{ selectedParent.ip
                          }})</template>
                          <template v-else>{{ selectedParent.ip }}</template>
                        </span>
                      </span>
                      <span v-else class="option-placeholder">-- 默认为根服务器 --</span>

                      <span class="custom-select-arrow" :class="{ 'is-open': isParentSelectOpen }">▼</span>
                    </button>

                    <transition name="modal-fade">
                      <ul v-if="isParentSelectOpen" class="custom-select-options is-parent-select"
                        ref="parentSelectOptionsRef" role="listbox">

                        <li class="custom-select-option" :class="{ 'is-selected': !currentServerData.parent_ip }"
                          @click="selectParent('')" role="option" :aria-selected="!currentServerData.parent_ip">
                          <span class="option-placeholder">-- 默认为根服务器 --</span>
                        </li>

                        <li v-for="parent in potentialParentServers" :key="parent.ip" class="custom-select-option"
                          :class="{
                            'is-selected': currentServerData.parent_ip === parent.ip,
                            'is-disabled': parent.ip === editingServerIp
                          }" @click="parent.ip === editingServerIp ? null : selectParent(parent.ip)" role="option"
                          :aria-selected="currentServerData.parent_ip === parent.ip"
                          :aria-disabled="parent.ip === editingServerIp">

                          <span v-if="parent.tag" class="simple-tag-small" :style="{
                            backgroundColor: parent.tag_color_with_hash,
                            color: getContrastColor(parent.tag_color_with_hash)
                          }">{{ parent.tag }}</span>

                          <span class="option-text">
                            <template v-if="parent.comment">{{ parent.comment }} ({{ parent.ip }})</template>
                            <template v-else>{{ parent.ip }}</template>
                          </span>
                        </li>
                      </ul>
                    </transition>

                  </div>
                  <p v-if="potentialParentServers.length === 0 && modalMode === 'add'" class="form-help-text">
                    当前没有可用的父服务器，将作为根服务器添加。
                  </p>
                </div>
              </div>

              <div class="form-row">
                <div class="form-group form-group-checkbox">
                  <input type="checkbox" v-model="currentServerData.ignore_in_list"
                    :id="'ignore_mod_' + sanitizeIpForId(currentServerData.ip || 'new')" class="styled-checkbox" />
                  <label :for="'ignore_mod_' + sanitizeIpForId(currentServerData.ip || 'new')">
                    <font-awesome-icon :icon="faEyeSlash" /> 在列表中隐藏 (ignore_in_list)
                  </label>
                </div>
              </div>

            </div>
          </div>

          <div class="modal-footer">
            <button @click="closeServerModal" class="btn-modal-cancel" type="button">
              <font-awesome-icon :icon="faTimes" /> 取消
            </button>
            <button @click="saveServer" class="btn-modal-confirm" type="button">
              <font-awesome-icon :icon="faSave" /> {{ modalMode === 'add' ? '确认添加' : '保存更改' }}
            </button>
          </div>

        </div>
      </div>
    </transition>

  </div>
</template>

<style>
/* --- (新增) 夜间模式 --- */

/* 1. 夜间模式的颜色变量 (覆盖 :root) */
html.dark-mode {
  --color-body-gradient-start: #111827;
  /* 深灰蓝 */
  --color-body-gradient-end: #1f2937;
  /* 稍浅的灰蓝 */

  --color-panel-gradient-start: #4f46e5;
  /* 调整渐变色 */
  --color-panel-gradient-end: #7c3aed;

  --color-surface: #1f2937;
  /* 卡片背景 */
  --color-surface-muted: #374151;
  /* 嵌套/次要背景 */

  --color-text-primary: #f3f4f6;
  /* 亮灰色 (非纯白) */
  --color-text-secondary: #9ca3af;
  /* 中灰色 */

  --color-border: #4b5563;
  /* 深色边框 */

  /* --- (v17.15) 提高对比度 --- */
  --color-primary: #818cf8;
  /* (原: #6366f1) 变亮 */
  --color-primary-hover: #6366f1;
  /* (原: #4f46e5) 使用旧的默认色 */
  --color-secondary: #60a5fa;
  /* (原: #3b82f6) 变亮 */
  --color-secondary-hover: #3b82f6;
  /* (原: #2563eb) 使用旧的默认色 */

  --color-focus-outline: rgba(129, 140, 248, 0.55);
  /* (v17.15) 匹配新的主色 */
  /* 提高不透明度 */

  --shadow-soft: 0 10px 20px rgba(0, 0, 0, 0.2);
  --shadow-hover: 0 16px 40px rgba(0, 0, 0, 0.25);
}

/* 2. 为 body 添加过渡动画 */
body {
  transition: background 0.3s ease;
}

/* 3. 为所有使用变量的元素添加过渡动画 */
.panel,
.modal-box,
.btn,
input[type="text"],
select,
textarea,
.color-picker,
.server-item-simple,
.btn-add-child-simple,
.btn-edit-simple,
.btn-modal-cancel,
.btn-modal-confirm,
.form-compound-input {
  transition: background 0.3s ease, color 0.3s ease, border-color 0.3s ease;
}

/* 4. 夜间模式切换按钮的样式 */
.panel-header {
  position: relative;
  /* 为按钮定位 */
}

.theme-toggle-btn {
  position: absolute;
  top: 20px;
  right: 20px;
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: #fff;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  font-size: 1.2rem;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

.theme-toggle-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

/* (夜间模式下切换按钮的样式) */
html.dark-mode .theme-toggle-btn {
  background: rgba(0, 0, 0, 0.2);
  color: #f3f4f6;
}

html.dark-mode .theme-toggle-btn:hover {
  background: rgba(0, 0, 0, 0.3);
}

/* --- (新增结束) --- */

:root {
  --color-body-gradient-start: #eef2ff;
  --color-body-gradient-end: #f9fafb;
  --color-panel-gradient-start: #6366f1;
  --color-panel-gradient-end: #7c3aed;
  --color-surface: #ffffff;
  --color-surface-muted: #f5f7fb;
  --color-text-primary: #1f2933;
  --color-text-secondary: #4b5563;
  --color-border: #dbe2ef;
  --color-primary: #4f46e5;
  --color-primary-hover: #4338ca;
  --color-secondary: #2563eb;
  --color-secondary-hover: #1d4ed8;
  --color-success: #10b981;
  --color-success-hover: #059669;
  --color-danger: #ef4444;
  --color-danger-hover: #dc2626;
  --color-focus-outline: rgba(99, 102, 241, 0.45);
  --shadow-soft: 0 10px 30px rgba(15, 23, 42, 0.08);
  --shadow-hover: 0 16px 40px rgba(79, 70, 229, 0.16);
}

/* 1. 全局和背景 */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  font-family: 'Segoe UI', 'PingFang SC', 'Microsoft YaHei', sans-serif;
}

body {
  background: linear-gradient(135deg, var(--color-body-gradient-start), var(--color-body-gradient-end));
  min-height: 100vh;
  padding: 20px;
  color: var(--color-text-primary);
  line-height: 1.5;
}

/* 2. 整体布局 */
.layout-container {
  display: grid;
  grid-template-columns: minmax(600px, 2fr) minmax(350px, 1fr);
  align-items: flex-start;
  gap: 20px;
  max-width: 1280px;
  width: 100%;
  margin: 0 auto;
}

@media (max-width: 1100px) {
  .layout-container {
    grid-template-columns: 1fr;
  }

  .io-panel {
    position: static;
    top: auto;
  }
}

/* 3. 面板样式 */
.panel {
  background: var(--color-surface);
  border-radius: 16px;
  box-shadow: var(--shadow-soft);
  overflow: hidden;
  min-width: 0;
  border: 1px solid var(--color-border);
}

.io-panel {
  position: sticky;
  top: 20px;
  align-self: flex-start;
}

.panel-header {
  background: linear-gradient(135deg, var(--color-panel-gradient-start), var(--color-panel-gradient-end));
  color: #fff;
  padding: 24px 20px;
  text-align: center;
}

.panel-header h1 {
  font-size: 28px;
  margin-bottom: 10px;
}

.panel-header .subtitle {
  font-size: 16px;
  opacity: 0.9;
}

.panel-body {
  padding: 20px;
}

/* 5. 表单和按钮 (用于模态框) */
/* (注意：这些是全局回退样式，新的弹窗将使用更具体的 .server-form 样式) */
.form-section {
  margin-bottom: 20px;
}

.form-section h3 {
  font-size: 1.4rem;
  color: var(--color-text-primary);
  margin-bottom: 15px;
  padding-bottom: 5px;
  border-bottom: 2px solid var(--color-border);
}

.io-panel .form-section h3 {
  font-size: 1.2rem;
  border-bottom: none;
  color: var(--color-primary);
}

.form-section p {
  margin-bottom: 10px;
  color: var(--color-text-secondary);
  font-size: 0.9rem;
}

.form-row {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 10px;
}

.form-group {
  margin-bottom: 0px;
  flex: 1;
  min-width: 100px;
  min-width: 0;
}

.form-group.grow {
  flex: 2;
}

label {
  display: block;
  margin-bottom: 3px;
  font-weight: 500;
  color: #2c3e50;
  font-size: 0.85rem;
}

/* v11 优化 #1: 必填项星号 */
label .required {
  color: var(--color-danger);
  font-weight: bold;
  margin-left: 2px;
}

input[type="text"],
select,
textarea {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid var(--color-border);
  border-radius: 10px;
  font-size: 0.95rem;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
  max-width: 100%;
  background-color: var(--color-surface);
  color: var(--color-text-primary);
}

input[type="text"]::placeholder,
textarea::placeholder {
  color: var(--color-text-secondary);
  opacity: 0.6;
}

input[type="text"]:focus,
select:focus,
textarea:focus {
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-focus-outline);
  outline: none;
}

button:focus-visible,
.btn:focus-visible,
.btn-close-modal:focus-visible {
  outline: 3px solid var(--color-focus-outline);
  outline-offset: 2px;
}

textarea {
  font-family: "JetBrains Mono", "Fira Code", "Consolas", "Courier New", monospace;
  font-size: 14px;
  line-height: 1.5;
  background-color: var(--color-surface);
  color: var(--color-text-primary);
}

.color-picker {
  height: 38px;
  padding: 4px;
  border-radius: 10px;
  border: 1px solid var(--color-border);
  width: 100%;
  background: var(--color-surface);
}

.form-group-checkbox {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 5px;
}

.form-group-checkbox label {
  margin-bottom: 0;
  font-weight: 400;
  color: #333;
  cursor: pointer;
}

.styled-checkbox {
  width: auto;
  height: 16px;
  width: 16px;
  accent-color: #4A00E0;
  cursor: pointer;
}

.select-wrapper {
  position: relative;
  width: 100%;
}

.select-wrapper select {
  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
  padding-right: 30px;
  width: 100%;
  cursor: pointer;
}

.select-wrapper select:disabled {
  background-color: #f1f1f1;
  color: #777;
  cursor: not-allowed;
}

.select-wrapper::after {
  content: '▼';
  font-size: 12px;
  color: #7e8c9a;
  position: absolute;
  right: 12px;
  top: 50%;
  transform: translateY(-50%);
  pointer-events: none;
}

/* (v16) 新增：表单辅助提示文本 */
.form-help-text {
  font-size: 0.8rem;
  color: #7e8c9a;
  margin-top: 5px;
  margin-bottom: 0;
}

.btn {
  border: 1px solid transparent;
  border-radius: 10px;
  padding: 10px 16px;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.95rem;
  font-family: inherit;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  white-space: nowrap;
  transition: background 0.2s ease, color 0.2s ease, box-shadow 0.2s ease, transform 0.2s ease, border-color 0.2s ease;
}

.btn:hover {
  transform: translateY(-1px);
  box-shadow: var(--shadow-hover);
}

.btn-primary {
  background: var(--color-success);
  color: #fff;
  border-color: transparent;
}

.btn-primary:hover {
  background: var(--color-success-hover);
}

.btn-secondary {
  background: var(--color-secondary);
  color: #fff;
  border-color: transparent;
}

.btn-secondary:hover {
  background: var(--color-secondary-hover);
}

.btn-danger {
  background: var(--color-danger);
  color: #fff;
  border-color: transparent;
}

.btn-danger:hover {
  background: var(--color-danger-hover);
}

.btn-add {
  background: var(--color-primary);
  color: #fff;
  border-color: transparent;
}

.btn-add:hover {
  background: var(--color-primary-hover);
}


/* 6. 服务器列表 */
.server-list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.header-actions {
  display: flex;
  gap: 10px;
}

.server-list {
  min-height: 50px;
}

.server-item-simple {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 12px 14px;
  display: flex;
  align-items: center;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.08);
  user-select: none;
}

.server-item-simple:hover {
  border-color: var(--color-primary);
  box-shadow: 0 10px 16px rgba(79, 70, 229, 0.15);
}

.server-item-simple.is-child {
  background: var(--color-surface-muted);
  border-left: 4px solid var(--color-panel-gradient-end);
  margin-bottom: 6px;
}

.server-item-simple.is-parent {
  border-left: 4px solid var(--color-panel-gradient-start);
}

.server-item-simple.is-ignored {
  opacity: 0.7;
}

.server-item-simple.is-ignored .simple-ip {
  text-decoration: line-through;
}

.simple-info {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 10px;
  overflow: hidden;
  white-space: nowrap;
  min-width: 0;
}

.simple-tag {
  font-size: 0.85rem;
  font-weight: 500;
  padding: 3px 8px;
  border-radius: 4px;
}

.simple-comment {
  font-weight: 500;
  color: var(--color-text-primary);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex-shrink: 1;
}

.simple-ip {
  font-family: "JetBrains Mono", "Consolas", monospace;
  font-size: 0.95rem;
  color: var(--color-text-primary);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex-shrink: 100;
}

.simple-ip.with-comment {
  color: var(--color-text-secondary);
  font-size: 0.85rem;
  margin-left: -5px;
}

.simple-ignored-badge {
  font-size: 0.8rem;
  font-weight: 500;
  color: var(--color-text-secondary);
}

.simple-actions {
  display: flex;
  gap: 8px;
  margin-left: 10px;
}

.btn-add-child-simple {
  background: var(--color-surface-muted);
  color: var(--color-primary);
  border: 1px solid var(--color-border);
  transition: background 0.2s ease, color 0.2s ease, border-color 0.2s ease;
}

.btn-add-child-simple:hover,
.btn-add-child-simple:focus-visible {
  background: var(--color-body-gradient-start);
  border-color: var(--color-primary);
  color: var(--color-primary);
}

.btn-edit-simple {
  background: var(--color-surface-muted);
  color: var(--color-primary);
  /* (v17.13) 保持颜色 */
  border: 1px solid var(--color-border);
  transition: background 0.2s ease, color 0.2s ease, border-color 0.2s ease;
}

.btn-edit-simple:hover,
.btn-edit-simple:focus-visible {
  background: var(--color-body-gradient-start);
  border-color: var(--color-primary);
}

.drag-handle {
  width: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-right: 12px;
  cursor: grab;
  color: var(--color-text-secondary);
  font-size: 1.5rem;
  padding-top: 0;
  user-select: none;
}

.drag-handle:active {
  cursor: grabbing;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: var(--color-text-secondary);
  /* (修正) */
  background: var(--color-surface-muted);
  /* (修正) */
  border-radius: 8px;
}

.parent-warning {
  color: #d32f2f;
  font-size: 0.85rem;
  margin-top: 5px;
}

.server-form {
  padding: 5px;
}

.server-item-simple.sortable-drag {
  opacity: 0.9;
  background: #f5f3ff;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  transform: scale(1.02);
}

.sortable-ghost {
  background: var(--color-body-gradient-start);
  border: 2px dashed var(--color-primary);
  opacity: 0.7;
  border-radius: 8px;
}

.sortable-ghost>* {
  visibility: hidden;
}

/* 7. 模态弹窗 (Alert/Confirm) */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.55);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  z-index: 2000;
}

.modal-overlay.edit-modal {
  z-index: 1000;
}

.modal-box {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 16px;
  box-shadow: var(--shadow-hover);
  width: min(90%, 520px);
  overflow: hidden;
  animation: modal-pop-in 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.modal-header {
  padding: 15px 20px;
  border-bottom: 1px solid var(--color-border);
  background: var(--color-surface-muted);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h3 {
  font-size: 1.25rem;
  color: var(--color-text-primary);
}

.modal-body {
  padding: 28px 24px;
  font-size: 1rem;
  line-height: 1.6;
  color: var(--color-text-primary);
}

.modal-body pre {
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  color: var(--color-text-secondary);
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding: 16px 24px;
  background: var(--color-surface-muted);
  border-top: 1px solid var(--color-border);
}

.btn-modal-cancel {
  background: var(--color-surface-muted);
  color: var(--color-text-primary);
  border: 1px solid var(--color-border);
  padding: 10px 22px;
  border-radius: 12px;
  font-weight: 600;
  font-size: 0.95rem;
  transition: background 0.2s ease, border-color 0.2s ease, color 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;
}

.btn-modal-cancel:hover,
.btn-modal-cancel:focus-visible {
  background: var(--color-body-gradient-start);
  border-color: var(--color-primary);
  color: var(--color-primary);
  transform: translateY(-1px);
  box-shadow: var(--shadow-hover);
}

.btn-modal-confirm {
  background: linear-gradient(135deg, var(--color-primary) 0%, var(--color-secondary) 100%);
  color: #fff;
  border: none;
  padding: 10px 26px;
  border-radius: 12px;
  font-weight: 600;
  font-size: 0.95rem;
  cursor: pointer;
  transition: background 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;
  box-shadow: 0 6px 16px rgba(79, 70, 229, 0.25);
}

.btn-modal-confirm:hover,
.btn-modal-confirm:focus-visible {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px rgba(79, 70, 229, 0.3);
  background: linear-gradient(135deg, var(--color-primary-hover) 0%, var(--color-secondary-hover) 100%);
}

@keyframes modal-pop-in {
  from {
    opacity: 0;
    transform: scale(0.8);
  }

  to {
    opacity: 1;
    transform: scale(1);
  }
}

.modal-fade-enter-active,
.modal-fade-leave-active {
  transition: opacity 0.3s ease;
}

.modal-fade-enter-from,
.modal-fade-leave-to {
  opacity: 0;
}

/* --- 编辑弹窗样式 --- */
.modal-overlay.edit-modal {
  z-index: 1500;
}

.modal-box.edit-modal-box {
  max-width: 760px;

  /* --- (v17.11) 新增：flex 布局 --- */
  display: flex;
  flex-direction: column;

  /* * 1. 使用 90dvh (动态视口高度) 来适应手机工具栏
   * 2. 90vh 是为不支持 dvh 的旧浏览器的后备 
   */
  max-height: 90vh;
  max-height: 90dvh;
}

.edit-modal-box .modal-header {
  background: linear-gradient(135deg, var(--color-panel-gradient-start), var(--color-panel-gradient-end));
  color: #fff;
  padding: 24px 28px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.18);
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-shrink: 0;
}

.edit-modal-box .modal-header h3 {
  font-size: 1.4rem;
  font-weight: 600;
  margin: 0;
}

.btn-close-modal {
  background: transparent;
  border: none;
  font-size: 1.5rem;
  line-height: 1;
  color: rgba(255, 255, 255, 0.85);
  cursor: pointer;
  padding: 4px 6px;
  border-radius: 50%;
  transition: background 0.2s ease, color 0.2s ease, transform 0.2s ease;
}

.btn-close-modal:hover {
  color: #fff;
  background: rgba(255, 255, 255, 0.18);
  transform: scale(1.08);
}

.edit-modal-box .modal-body {
  padding: 28px;
  overflow-y: auto;
  background: var(--color-surface);
  scrollbar-gutter: stable;

  flex-grow: 1;
  min-height: 0;
}

.server-form {
  padding: 4px 0;
}

.server-form .form-row {
  display: flex;
  flex-wrap: wrap;
  gap: 18px;
  margin-bottom: 18px;
}

.server-form .form-group {
  flex: 1;
  min-width: 160px;
  position: relative;
}

.server-form label {
  display: flex;
  align-items: center;
  margin-bottom: 8px;
  font-weight: 600;
  color: var(--color-text-primary);
  font-size: 0.95rem;
}

.server-form .required {
  color: var(--color-danger);
  margin-left: 4px;
}

.server-form input[type="text"],
.server-form select {
  width: 100%;
  padding: 12px 14px;
  border: 1px solid var(--color-border);
  border-radius: 12px;
  background: var(--color-surface);
  box-shadow: 0 2px 6px rgba(15, 23, 42, 0.06);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
  padding-left: 26px;
  color: var(--color-text-primary);
  position: relative;
  /* <-- (v17.13) 新增 */
  z-index: 1;
  /* <-- (v17.13) 新增 */
}

.server-form input[type="text"]:focus,
.server-form select:focus {
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-focus-outline);
  outline: none;
}

.server-form .color-picker {
  height: 48px;
  padding: 4px;
  border-radius: 12px;
  border: 1px solid var(--color-border);
  width: 100%;
  cursor: pointer;
  background: var(--color-surface);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.server-form .color-picker:focus {
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-focus-outline);
  outline: none;
}

.server-form .form-group-checkbox {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 10px;
  padding: 10px 0;
}

.server-form .form-group-checkbox label {
  margin-bottom: 0;
  font-weight: 500;
  color: var(--color-text-primary);
  cursor: pointer;
  font-size: 0.95rem;
}

.server-form .styled-checkbox {
  width: 20px;
  height: 20px;
  accent-color: var(--color-primary);
  cursor: pointer;
}

.server-form .select-wrapper::after {
  right: 15px;
}

.server-form .form-help-text {
  font-size: 0.85rem;
  color: var(--color-text-secondary);
  margin-top: 8px;
  margin-bottom: 0;
  line-height: 1.4;
}

.edit-modal-box .modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 15px;
  padding: 18px 28px;
  background: var(--color-surface-muted);
  border-top: 1px solid var(--color-border);
  flex-shrink: 0;
}

.server-form .form-group::before {
  content: '';
  position: absolute;
  top: 42px;
  left: 12px;
  width: 4px;
  height: 20px;
  background: linear-gradient(to bottom, var(--color-panel-gradient-start), var(--color-panel-gradient-end));
  border-radius: 2px;
  opacity: 0.6;
  z-index: 2;
}

.server-form .form-group:has(> .color-picker)::before,
.server-form .form-group-checkbox::before {
  display: none;
}

.server-form .form-group:has(.color-picker) input {
  padding-left: 4px;
}

@media (max-width: 440px) {
  .server-form .form-row {
    flex-direction: column;
    gap: 15px;
  }

  .edit-modal-box {
    width: 95%;
  }
}

/* --- 弹窗美化 - 结束 --- */


/* 9. (v14/v15) 拖拽容器和子列表样式 (原始代码，保持不变) */
.server-item-container {
  margin-bottom: 6px;
  position: relative;
}

.child-list {
  margin-left: 40px;
}

.server-item-container.is-parent-container .child-list {
  margin-top: 6px;
  min-height: 20px;
}

.server-item-container:not(.is-parent-container) .child-list {
  margin-top: 0;
  min-height: 0;
}

/* (v16 建议 1)
 * 减小子列表 "幽灵" (占位符) 的最小高度
 * 这使得意外拖入子列表的难度增加
 */
.child-list.sortable-ghost {
  min-height: 30px;
  /* <-- 原为 50px */
  background: var(--color-body-gradient-start);
  border: 2px dashed var(--color-primary);
  border-radius: 8px;
}

.child-list.sortable-ghost>* {
  visibility: hidden;
}

.server-list-anim-root-enter-from,
.server-list-anim-root-leave-to {
  opacity: 0;
  transform: scale(0.9);
}

.server-list-anim-root-enter-active,
.server-list-anim-root-leave-active {
  transition: all 0.3s ease;
}

.server-list-anim-root-move {
  transition: transform 0.3s cubic-bezier(0.55, 0, 0.1, 1);
}

.server-item-container.sortable-ghost {
  background: var(--color-body-gradient-start);
  border: 2px dashed var(--color-primary);
  opacity: 0.7;
  border-radius: 8px;
  min-height: 50px;
}

.server-item-container.sortable-ghost>* {
  visibility: hidden;
}

.server-list-anim-child-enter-active,
.server-list-anim-child-leave-active {
  transition: all 0.3s ease;
}

.server-list-anim-child-enter-from,
.server-list-anim-child-leave-to {
  opacity: 0;
  transform: translateX(30px);
}

.server-list-anim-child-move {
  transition: transform 0.3s ease;
}

/* 1. 复合框的容器 */
.form-compound-input {
  display: flex;
  align-items: center;
  position: relative;
  width: 100%;
  border: 1px solid var(--color-border);
  border-radius: 12px;
  background: var(--color-surface);
  box-shadow: 0 2px 6px rgba(15, 23, 42, 0.06);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.form-compound-input:focus-within {
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-focus-outline);
}

/* 2. 内部的文本输入框 */
.form-compound-input-text {
  flex: 1 1 auto;
  border: none !important;
  outline: none !important;
  box-shadow: none !important;
  background: transparent;
  padding-left: 26px !important;
  min-width: 0;
}

/* * 3. (已修正) 内部的颜色选择器
 * 我们添加了 .server-form 前缀来提高优先级
*/
.server-form .form-compound-input-color {
  flex: 0 0 auto;
  width: 44px;
  /* <--- 现在这个会生效了 */
  height: 44px;
  /* <--- 这个也会生效 */
  margin: 2px;
  border: 1px solid var(--color-border);
  border-radius: 10px;
  padding: 4px;
}

/* (已修正) 同样为 :focus 规则添加前缀 */
.server-form .form-compound-input-color:focus {
  outline: none;
  box-shadow: none;
}

.custom-select-container {
  position: relative;
  width: 100%;
}

.custom-select-trigger {
  /* 模拟 .server-form input[type="text"] 样式 */
  width: 100%;
  padding: 12px 14px;
  border: 1px solid var(--color-border);
  border-radius: 12px;
  background: var(--color-surface);
  box-shadow: 0 2px 6px rgba(15, 23, 42, 0.06);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;

  /* 按钮特定样式 */
  display: flex;
  align-items: center;
  justify-content: space-between;
  text-align: left;
  cursor: pointer;
  font-size: 0.95rem;
  /* 匹配 input */
  color: var(--color-text-primary);
  /* 匹配 input */

  /* 模拟 :focus 样式 */
  outline: none;

  /* 模拟 input padding-left: 26px (已由 ::before 伪元素占用) */
  padding-left: 26px;
  position: relative;
  /* <-- (v17.13) 新增 */
  z-index: 1;
  /* <-- (v17.13) 新增 */
}

.custom-select-trigger:focus-visible {
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px var(--color-focus-outline);
}

.custom-select-trigger:disabled {
  background-color: var(--color-surface-muted);
  color: var(--color-text-secondary);
  cursor: not-allowed;
  opacity: 0.7;
}

/* 触发器内部布局 */
.selected-option-content {
  display: flex;
  align-items: center;
  gap: 8px;
  overflow: hidden;
  /* 关键：防止内容溢出 */
}

.option-text {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.option-placeholder {
  color: var(--color-text-secondary);
  opacity: 0.8;
}

.custom-select-arrow {
  font-size: 12px;
  color: var(--color-text-secondary);
  transition: transform 0.2s ease;
  transform-origin: center;
  margin-left: 8px;
  /* 增加一点间距 */
  flex-shrink: 0;
}

.custom-select-arrow.is-open {
  transform: rotate(180deg);
}

/* 选项列表 */
.custom-select-options {
  position: absolute;
  top: 105%;
  /* 紧贴在触发器下方 */
  left: 0;
  right: 0;
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  box-shadow: var(--shadow-soft);
  z-index: 1600;
  /* 必须高于 modal-body (1500) */
  max-height: 200px;
  overflow-y: auto;
  list-style: none;
  padding: 5px;
  margin: 0;
  overscroll-behavior-y: contain;
}

/* 单个选项 */
.custom-select-option {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.2s ease;
  color: var(--color-text-primary);
  overflow: hidden;
  /* 确保内容不溢出 */
}

.custom-select-option:hover {
  background: var(--color-surface-muted);
}

.custom-select-option.is-selected {
  background: var(--color-body-gradient-start);
  color: var(--color-primary);
  font-weight: 600;
}

.custom-select-option.is-disabled {
  color: var(--color-text-secondary);
  opacity: 0.6;
  cursor: not-allowed;
  background: transparent;
}

.custom-select-option.is-disabled:hover {
  background: transparent;
}

/* 内部小标签 (用于触发器和选项) */
.simple-tag-small {
  font-size: 0.8rem;
  font-weight: 500;
  padding: 2px 6px;
  border-radius: 4px;
  flex-shrink: 0;
  /* 防止标签被压缩 */
  /* 其他样式 (背景色/颜色) 由内联 :style 提供 */
}

.custom-select-options.is-parent-select {
  top: auto;
  bottom: 105%;
  box-shadow: var(--shadow-soft);
  /* (可选) 确保阴影在上方也好看 */
}

/* --- (v17.4) 1. 自定义夜间模式滚动条 --- */
html.dark-mode {
  /* 适用于 Firefox */
  scrollbar-color: #6b7280 #374151;
  scrollbar-width: thin;
}

html.dark-mode ::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

html.dark-mode ::-webkit-scrollbar-track {
  background: var(--color-surface-muted);
  /* 滚动条轨道背景 */
  border-radius: 4px;
}

html.dark-mode ::-webkit-scrollbar-thumb {
  background-color: #6b7280;
  /* 滚动条滑块 */
  border-radius: 4px;
  /* (可选) 创建一个“内边距”效果，让滑块看起来更细 */
  border: 2px solid var(--color-surface-muted);
}

html.dark-mode ::-webkit-scrollbar-thumb:hover {
  background-color: var(--color-text-secondary);
  /* 悬停时变亮 */
}

/* --- (v17.4) 1. 自定义夜间模式滚动条 --- */
html.dark-mode {
  /* 适用于 Firefox */
  scrollbar-color: #6b7280 #374151;
  scrollbar-width: thin;
}

html.dark-mode ::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

html.dark-mode ::-webkit-scrollbar-track {
  background: var(--color-surface-muted);
  /* 滚动条轨道背景 */
  border-radius: 4px;
}

html.dark-mode ::-webkit-scrollbar-thumb {
  background-color: #6b7280;
  /* 滚动条滑块 */
  border-radius: 4px;
  /* (可选) 创建一个“内边距”效果，让滑块看起来更细 */
  border: 2px solid var(--color-surface-muted);
}

html.dark-mode ::-webkit-scrollbar-thumb:hover {
  background-color: var(--color-text-secondary);
  /* 悬停时变亮 */
}

/* --- (v17.6) 微调 Font Awesome SVG 图标对齐 --- */

/* * 目标：按钮和标签中的图标
 * 作用：确保图标与旁边的文本垂直居中 
 */
.btn svg,
.btn-modal-cancel svg,
.btn-modal-confirm svg,
.form-group-checkbox label svg,
.modal-header h3 svg {
  /* * 这是一个魔法数字，用于将 SVG 稍微向下移动一点
   * 使其在视觉上与文本的“中线”对齐 
   */
  vertical-align: -0.125em;

  /* (可选) 如果图标和文本贴得太近，请取消注释此行 */
  /* margin-right: 0.3em; */
}

/* * 目标：单独的图标 (例如夜间模式、关闭按钮) 
 * 作用：确保它们正确地填充其容器 
 */
.theme-toggle-btn svg,
.btn-close-modal svg {
  vertical-align: middle;
}

/* --- (v17.7) 自定义颜色选择器样式 --- */
.color-picker-wrapper {
  /* * 1. 模拟 .form-compound-input-color 的大小和位置 
   * (flex: 0 0 auto; width: 44px; height: 44px; margin: 2px;)
   */
  flex: 0 0 auto;
  position: relative;
  width: 44px;
  height: 44px;
  margin: 2px;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1;
  /* <-- (v17.13) 新增 */
}

.custom-color-trigger {
  width: 36px;
  height: 36px;
  border: 1px solid var(--color-border);
  border-radius: 8px;
  /* 稍小的圆角 */
  cursor: pointer;
  padding: 0;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.custom-color-trigger:hover {
  transform: scale(1.1);
}

.custom-color-trigger:focus-visible {
  outline: 3px solid var(--color-focus-outline);
  outline-offset: 2px;
}

.color-picker-modal-overlay {
  /* 1. 复制 .modal-overlay 的样式 */
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.55);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;

  /* 2. 确保它在编辑弹窗 (1500) 和下拉框 (1600) 之上 */
  z-index: 1800;
}

.color-picker-modal-box {
  /* 1. 复制 .modal-box 的样式 */
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 16px;
  box-shadow: var(--shadow-hover);

  /* 2. 自定义样式 */
  padding: 12px;
  /* 紧凑一点 */
  animation: modal-pop-in 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  overscroll-behavior-y: contain;
}

/* 3. (重要) 覆盖 vue3-colorpicker 的默认样式 */
.color-picker-modal-box .vc-color-wrap {
  border: none !important;
  box-shadow: none !important;
  background: transparent !important;
  width: 280px;
  /* (v17.10) 给予一个在手机上刚好的固定宽度 */
}

.color-picker-modal-box .vc-color-wrap.is-widget {
  padding: 0 !important;
}

/* 4. "完成" 按钮的样式 */
.btn-color-picker-done {
  width: 100%;
  /* 占满宽度 */
  margin-top: 10px;
  font-size: 1rem;
  padding-top: 12px;
  padding-bottom: 12px;
}

/* --- (v17.13) 新增：统一的图标按钮样式 --- */
.btn-icon-simple {
  /* 1. 统一大小和形状 */
  width: 32px;
  height: 32px;
  padding: 0;
  font-size: 0.9rem;
  /* 统一图标大小 */
  border-radius: 8px;

  /* 2. 居中图标 */
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

/* 3. 确保悬停时样式一致 (无 transform) */
.btn-icon-simple:hover,
.btn-icon-simple:focus-visible {
  transform: none;
  box-shadow: none;
}

/* 4. 将新样式应用到 "X" 按钮 */
/* 4. 将新样式应用到删除按钮 */
.btn-remove-simple {
  background: var(--color-surface-muted);
  border: 1px solid var(--color-border);
  color: var(--color-danger);
  /* <-- (新增) 默认状态即为红色 */
}

.btn-remove-simple:hover,
.btn-remove-simple:focus-visible {
  background: var(--color-danger);
  /* <-- (修改) 悬停时背景变红 */
  border-color: var(--color-danger-hover);
  color: #fff;
  /* <-- (修改) 悬停时图标变白 */
}

/* --- (v17.22) 移动端适配：使用 HTML 包装器 --- */

@media (max-width: 600px) {

  /* 1. (不变) 父容器顶部对齐 */
  .server-item-simple {
    padding: 10px 10px 12px 10px;
    align-items: flex-start;
  }

  /* 2. (核心) 信息区：
   * 1. 允许换行
   * 2. (关键) 为按钮留出空间
   */
  .simple-info {
    flex-wrap: wrap;
    /* 允许换行 */
    white-space: normal;
    overflow: hidden;
    gap: 4px 10px;
    align-items: center;
    /* (关键) 为 3 个按钮 (3*32px) + 2 个间隙 (2*5px) = 106px */
    width: calc(100% - 106px - 24px - 10px);
    /* 100% - 按钮 - 拖拽柄 - 间隙 */
  }

  /* 3. (核心) 第二行包裹器：
   * 1. 强制换行
   * 2. 内部使用 flex 布局
   */
  .simple-info-line2 {
    flex-basis: 100%;
    /* (关键) 强制到第二行 */
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
    /* 允许内部元素收缩 */
  }

  /* --- 4. 第一行项目 --- */

  .simple-tag {
    white-space: nowrap;
    flex-shrink: 0;
  }

  .simple-comment {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    flex-shrink: 1;
    min-width: 0;
  }

  /* --- 5. 第二行项目 --- */

  .simple-ip {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    flex-shrink: 1;
    /* (关键) 允许 IP 收缩 */
    min-width: 0;
  }

  .simple-ignored-badge {
    white-space: nowrap;
    flex-shrink: 0;
    /* (关键) "已隐藏" 不收缩 */
  }

  /* --- 6. (不变) 修复样式和拖拽柄 --- */

  .simple-ip.with-comment {
    margin-left: 0;
    font-size: 0.85rem;
  }

  .drag-handle {
    margin-top: 1px;
  }

  .simple-actions {
    gap: 5px;
    margin-left: 8px;
    /* 恢复 margin */
    padding-top: 0;
  }

  /* --- (v17.23) 修复服务器列表标题换行 --- */

  .server-list-header {
    flex-wrap: wrap;
    /* 1. 允许标题和按钮组换行 */
    gap: 10px 15px;
    /* 2. 换行后的垂直/水平间距 */
  }

  .server-list-header h3 {
    white-space: nowrap;
    /* 3. 确保 "服务器列表" 5个字不换行 */
    font-size: 1.3rem;
    /* 4. 稍微缩小字体 */
    margin-bottom: 0;
    /* 5. 移除 h3 上的间距，使用 gap 代替 */
  }

  .header-actions {
    flex-grow: 1;
    /* 6. (可选) 让按钮组在换行时占满宽度 */
    justify-content: flex-end;
    /* 7. (可选) 让按钮保持在右侧 */
  }

}
</style>
