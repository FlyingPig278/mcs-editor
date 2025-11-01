<script setup lang="ts">
// --- 1. 导入依赖 ---
import { ref, computed, nextTick, watch, onMounted, onBeforeUnmount, type Ref } from 'vue'
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
  faClipboard, // <-- (v17.15) 新增
  faSpinner
} from '@fortawesome/free-solid-svg-icons'
import { ColorPicker } from 'vue3-colorpicker'
import 'vue3-colorpicker/style.css'
import pako from 'pako'

// (优化) 定义服务器数据在紧凑数组中的位置
const S_IP = 0;
const S_COMMENT = 1;
const S_TAG = 2;
const S_TAG_COLOR = 3;
const S_IGNORE = 4;
const S_CHILDREN = 5;

// (优化) 将清理过的 JSON 树转换为紧凑数组
function jsonToCompactArray(servers: Server[]): any[] {
  if (!servers) return [];
  return servers.map(server => {
    const children = (server.children && server.children.length > 0)
      ? jsonToCompactArray(server.children)
      : 0;

    return [
      server.ip,
      server.comment || 0,
      server.tag || 0,
      server.tag_color || 0,
      server.ignore_in_list ? 1 : 0,
      children
    ];
  });
}

// (优化) 将紧凑数组转换回 JSON 树
function compactArrayToJson(compactData: any[]): Server[] {
  return compactData.map(item => {
    const children = Array.isArray(item[S_CHILDREN])
      ? compactArrayToJson(item[S_CHILDREN])
      : [];

    const server: Server = {
      ip: item[S_IP],
      comment: item[S_COMMENT] || '',
      tag: item[S_TAG] || '',
      tag_color: item[S_TAG_COLOR] || 'FF9800',
      tag_color_with_hash: item[S_TAG_COLOR] ? `#${item[S_TAG_COLOR]}` : '#FF9800',
      ignore_in_list: item[S_IGNORE] === 1,
      children: children,
      // 以下是需要补全的默认/计算属性
      server_type: 'standalone', // 将在 buildTree 中被修正
      parent_ip: '', // 将在 buildTree 中被修正
      selectedPreset: ''
    };
    return server;
  });
}


// --- (优化) Zlib + Base64 编解码 --- 

function toUrlSafeBase64(base64: string): string {
  return base64.replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
}

function fromUrlSafeBase64(urlSafeBase64: string): string {
  let base64 = urlSafeBase64.replace(/-/g, '+').replace(/_/g, '/');
  while (base64.length % 4) {
    base64 += '=';
  }
  return base64;
}

function compressConfig(configObject: AppConfig): string {
  try {
    const compactServers = jsonToCompactArray(configObject.servers);
    const compactStructure = [
      configObject.footer || 0,
      configObject.show_offline_by_default ? 1 : 0,
      compactServers
    ];
    const jsonString = JSON.stringify(compactStructure);
    const compressed = pako.deflate(jsonString, { level: 9 });
    const base64 = btoa(String.fromCharCode.apply(null, compressed as any));
    return toUrlSafeBase64(base64);
  } catch (e) {
    console.error("压缩失败:", e);
    return '';
  }
}

function decompressConfig(encodedString: string): AppConfig | null {
  try {
    const base64 = fromUrlSafeBase64(encodedString);
    const binaryString = atob(base64);
    const charData = binaryString.split('').map(x => x.charCodeAt(0));
    const inflated = pako.inflate(new Uint8Array(charData), { to: 'string' });
    const compactStructure = JSON.parse(inflated);

    const servers = compactArrayToJson(compactStructure[2]);

    return {
      footer: compactStructure[0] === 0 ? '' : (compactStructure[0] || ''),
      show_offline_by_default: compactStructure[1] === 1,
      servers: servers
    };
  } catch (e) {
    console.error("解压失败:", e);
    return null;
  }
}

// --- (TS) 核心数据结构定义 ---
interface Server {
  ip: string;
  comment: string;
  tag: string;
  tag_color: string;
  tag_color_with_hash: string;
  server_type: 'standalone' | 'parent' | 'child';
  parent_ip: string;
  selectedPreset: string;
  ignore_in_list: boolean;
  priority?: number; // 拖拽时生成
  children: Server[]; // 用于 serverTree
}

interface AppConfig {
  footer: string;
  servers: Server[];
  show_offline_by_default: boolean;
}


// --- 2. 静态配置数据 ---

/**
 * @description 预设服务器标签
 */
const presets: Record<string, { tag: string; tag_color_with_hash: string }> = {
  "lobby": { tag: "大厅", tag_color_with_hash: "#3498DB" },
  "survival": { tag: "生存", tag_color_with_hash: "#2ECC71" },
  "creative": { tag: "创造", tag_color_with_hash: "#F1C40F" },
  "mod": { tag: "模组", tag_color_with_hash: "#E67E22" },
  "pvp": { tag: "PVP", tag_color_with_hash: "#E74C3C" },        // <-- 新增：PVP (战斗红)
  "recreation": { tag: "复原", tag_color_with_hash: "#9B59B6" } // <-- 新增：复原 (高校紫)
}

/**
 * @description (v16 重构) serverTypes 和 labels 仅用于显示
 */
const serverTypeLabels: Record<string, string> = {
  standalone: '独立服务器',
  parent: '父服务器',
  child: '子服务器'
}

// (v16 建议 5) 移除 defaultJsonString

// --- 3. 核心响应式状态 ---

const config = ref<AppConfig>({
  footer: "",
  servers: [],
  show_offline_by_default: false // (v17.34) 新增：全局配置
})

const serverTree = ref<Server[]>([])

const jsonInput = ref(``)

// (v17.15) 主题切换
const isDarkMode = ref(false);

function applyTheme(dark: boolean) {
  isDarkMode.value = dark;
  if (dark) {
    document.documentElement.classList.add('dark-mode');
    localStorage.setItem('theme', 'dark');
  } else {
    document.documentElement.classList.remove('dark-mode');
    localStorage.setItem('theme', 'light');
  }
}

function toggleTheme() {
  applyTheme(!isDarkMode.value);
}

// (我们将在下面的 onMounted 中调用它)

// --- 4. 模态弹窗状态 ---

// --- (A) 通用 Alert/Confirm 弹窗 ---
const isModalVisible = ref(false)
const modalTitle = ref('')
const modalMessage = ref('')
const modalType = ref('alert')
const modalResolve = ref<((value: boolean | PromiseLike<boolean>) => void) | null>(null)

// --- (B) (v16 重构) 统一的“添加/编辑”服务器弹窗状态 ---
const isServerModalVisible = ref(false) // 控制新弹窗的显示与隐藏
const modalMode = ref<'add' | 'edit'>('add') // 'add' 或 'edit'
const currentServerData = ref<Partial<Server> | null>(null) // 存储正在添加/编辑的服务器数据 (副本)
const editingServerIp = ref<string | null>(null) // 存储原始 IP，用于编辑时的唯一性检验
const isSaving = ref(false)             // (优化) 保存按钮的加载状态
const alertModalRef = ref<HTMLElement | null>(null)         // Alert/Confirm 弹窗容器
const serverModalRef = ref<HTMLElement | null>(null)        // 服务器编辑弹窗容器

// (优化) 自定义下拉框 A11y 状态
const isParentSelectOpen = ref(false)
const activeParentIndex = ref(-1)

const parentSelectTriggerRef = ref<HTMLElement | null>(null) // 触发器按钮
const parentSelectOptionsRef = ref<HTMLElement | null>(null) // 选项列表
const isPresetSelectOpen = ref(false)
const presetSelectTriggerRef = ref<HTMLElement | null>(null)
const presetSelectOptionsRef = ref<HTMLElement | null>(null)
const activePresetIndex = ref(-1)
const isColorPickerOpen = ref(false)
const colorPickerTriggerRef = ref<HTMLElement | null>(null)
const colorPickerPanelRef = ref<HTMLElement | null>(null)
let cancelModalTimeout: number | null = null;           // 统一的弹窗清理句柄

// (优化) Toast 小提示状态
const isToastVisible = ref(false);
const toastMessage = ref('');
let toastTimeout: number | null = null;

// (优化) 使用常量替代魔法字符串，提高代码可维护性
const MODAL_MODE = Object.freeze({ ADD: 'add', EDIT: 'edit' });
const SERVER_TYPE = Object.freeze({ STANDALONE: 'standalone', PARENT: 'parent', CHILD: 'child' });

const modalIds = {
  alertTitle: 'alert-modal-title',
  alertMessage: 'alert-modal-message',
  serverTitle: 'server-modal-title',
  serverBody: 'server-modal-body'
}

const focusableSelectors = 'a[href], button:not([disabled]), textarea:not([disabled]), input:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex^="-"])'

function focusFirstWithin(containerRef: Ref<HTMLElement | null> | HTMLElement) {
  nextTick(() => {
    const container = containerRef instanceof HTMLElement ? containerRef : containerRef?.value;
    if (!container) return
    const preferred = container.querySelector('[data-autofocus]') as HTMLElement | null
    const fallback = container.querySelector(focusableSelectors) as HTMLElement | null
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

function trapFocus(event: KeyboardEvent, containerRef: Ref<HTMLElement | null> | HTMLElement) {
  const container = containerRef instanceof HTMLElement ? containerRef : containerRef?.value;
  if (!container) return
  const focusable = Array.from(container.querySelectorAll(focusableSelectors)) as HTMLElement[];
  if (focusable.length === 0) {
    event.preventDefault()
    container.focus()
    return
  }
  const first = focusable[0]
  const last = focusable[focusable.length - 1]
  const active = document.activeElement as HTMLElement | null;
  if (event.shiftKey) {
    if (active === first || active === container) {
      event.preventDefault()
      last?.focus()
    }
  } else if (active === last) {
    event.preventDefault()
    first?.focus()
  }
}

function handleGlobalKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape') {
    if (isModalVisible.value) { // ✅ 优先检查 z-index 最高的弹窗
      event.preventDefault()
      onModalCancel()
    }
    // --- (v17.7 新增) 其次检查颜色选择器 ---
    else if (isColorPickerOpen.value) {
      event.preventDefault()
      isColorPickerOpen.value = false;
      (colorPickerTriggerRef.value as HTMLElement | null)?.focus()
    }
    // --- 结束 ---
    else if (isPresetSelectOpen.value) { // (v17.2) 其次检查预设下拉框
      event.preventDefault()
      isPresetSelectOpen.value = false;
      (presetSelectTriggerRef.value as HTMLElement | null)?.focus()
    }
    else if (isParentSelectOpen.value) { // (v17) 其次检查父服下拉框
      event.preventDefault()
      isParentSelectOpen.value = false;
      (parentSelectTriggerRef.value as HTMLElement | null)?.focus() // 焦点返回触发器
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
const potentialParentServers = computed<Server[]>(() => {
  if (!config.value.servers) return []
  return config.value.servers.filter(s => s.server_type === SERVER_TYPE.PARENT || s.server_type === SERVER_TYPE.STANDALONE)
})

/**
 * @description 计算出最终用于“导出”的 JSON 字符串。
 * (v_Fix) 修改为导出 'serverTree' (嵌套结构) 以方便后端渲染。
 */
const outputJson = computed<AppConfig>(() => {
  // 1. 创建一个干净的、只包含导出所需数据的服务器树
  function cleanNode(node: Server): Server {
    const newNode: Server = {
      ip: node.ip,
      comment: node.comment,
      tag: node.tag,
      tag_color: node.tag_color,
      tag_color_with_hash: node.tag_color_with_hash,
      ignore_in_list: node.ignore_in_list,
      children: [], // Default to empty array
      // Fill in other required Server properties with defaults
      server_type: node.server_type || 'standalone',
      parent_ip: node.parent_ip || '',
      selectedPreset: node.selectedPreset || ''
    };
    if (node.children && node.children.length > 0) {
      newNode.children = node.children.map(cleanNode);
    }
    return newNode;
  }

  const cleanServers = serverTree.value.map(cleanNode);

  // 2. 构建最终的配置对象
  return {
    footer: config.value.footer,
    show_offline_by_default: config.value.show_offline_by_default,
    servers: cleanServers
  };
});

/**
 * @description (v15) 拖拽规则。
 */
function checkMove(moveEvent: any): boolean {
  const draggedEl = moveEvent.draggedContext.element as Server
  const toEl = moveEvent.to as HTMLElement

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

function updateColorFromPicker(server: Server) {
  server.tag_color = server.tag_color_with_hash.substring(1).toUpperCase()
}

function applyPreset(server: Server) {
  const presetKey = server.selectedPreset
  if (!presetKey || !presets[presetKey]) return

  const preset = presets[presetKey]
  if (preset) {
    server.tag = preset.tag
    server.tag_color_with_hash = preset.tag_color_with_hash
    updateColorFromPicker(server)
  }
}

function checkIfCustom(server: Server) {
  if (!server.selectedPreset || !presets[server.selectedPreset]) return
  const preset = presets[server.selectedPreset]

  if (preset && (server.tag !== preset.tag || server.tag_color_with_hash !== preset.tag_color_with_hash)) {
    server.selectedPreset = ""
  }
}

function onColorInput(server: Server) {
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
const addChildServer = (parent: Server) => openServerModal(null, parent)

/**
 * @description 删除一个服务器（及其所有子服务器）。
 */
async function removeServer(server: Server) {
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

/**
 * @description (优化) 计算出用于 URL 分享的压缩字符串
 */
const compressedOutput = computed<string>(() => {
  return compressConfig(outputJson.value);
});

/**
 * @description 计算出用于QQ机器人的导入命令
 */
const importCommand = computed<string>(() => {
  return `/mcs import ${compressedOutput.value}`;
});

/**
 * @description 复制QQ机器人导入命令
 */
function copyImportCommand() {
  navigator.clipboard.writeText(importCommand.value).then(() => {
    showToast('导入命令已复制！请直接将其发送到QQ群中。');
  }, () => {
    showAlert('复制失败！请检查浏览器权限。', '复制失败');
  });
}

function downloadJson() {
  const jsonString = JSON.stringify(outputJson.value, null, 2);
  const blob = new Blob([jsonString], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'mcs-config.json';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

/**
 * @description (v17) 计算当前选中的父服务器对象，用于自定义下拉框显示
 */
const selectedParent = computed<Server | undefined>(() => {
  if (!currentServerData.value || !currentServerData.value.parent_ip) {
    return undefined
  }
  return potentialParentServers.value.find(p => p.ip === currentServerData.value?.parent_ip)
})

/**
 * @description (v17.2) 计算当前选中的预设对象，用于自定义下拉框显示
 */
const selectedPresetObject = computed<{ tag: string; tag_color_with_hash: string; } | undefined>(() => {
  if (!currentServerData.value || !currentServerData.value.selectedPreset) {
    return undefined
  }
  return presets[currentServerData.value.selectedPreset]
})

// --- 6. 方法 (Methods) ---

// --- (A) 通用 Alert/Confirm 弹窗方法 ---

function showAlert(message: string, title = '提示') {
  modalTitle.value = title
  modalMessage.value = message
  modalType.value = 'alert'
  isModalVisible.value = true
}

function showConfirm(message: string, title = '请确认'): Promise<boolean> {
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
    (modalResolve.value as (value: boolean) => void)(true)
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

const isAddMode = computed(() => modalMode.value === MODAL_MODE.ADD);

// (v17.34) defineExpose, 解决模板引用问题
defineExpose({
  // Data
  config,
  jsonInput,
  outputJson,
  compressedOutput,
  importCommand,
  isModalVisible,
  modalMode,
  currentServerData,
  isPresetSelectOpen,
  isParentSelectOpen,
  potentialParentServers,
  selectedParent,
  selectedPresetObject,
  isSaving,
  isColorPickerOpen,

  // Computed
  isAddMode,

  // Methods
  applyTheme,
  toggleTheme,
  buildTree,
  flattenTreeAndSync,
  parseAndSetConfig,
  addServer,
  removeServer,
  addChildServer,
  openServerModal,
  closeServerModal,
  saveServer,
  checkMove,
  handleGlobalKeydown,
  showConfirm,
  showAlert,
  showToast,
  copyImportCommand,
  downloadJson,
  loadConfig,
  onColorInput,
  checkIfCustom,
  sanitizeIpForId,
  removeAllServers,
  handleClickOutsideParentSelect,
  handleClickOutsidePresetSelect,
  toggleParentSelect,
  togglePresetSelect,
  selectParent,
  selectPreset,
  handleParentSelectKeydown,
  handlePresetSelectKeydown,
  onModalConfirm,
  onModalCancel
});

/**
 * @description (优化) 显示一个短暂的 Toast 提示
 * @param message - 要显示的消息
 */
function showToast(message: string) {
  // 如果当前有提示正在显示，先清除旧的计时器
  if (toastTimeout) {
    clearTimeout(toastTimeout);
  }

  toastMessage.value = message;
  isToastVisible.value = true;

  // 3秒后自动隐藏
  toastTimeout = window.setTimeout(() => {
    isToastVisible.value = false;
    toastTimeout = null;
  }, 3000);
}

// --- (B) (v16 重构) 统一的“添加/编辑”服务器弹窗方法 ---

/**
 * @description 创建一个用于“添加”弹窗的空白服务器对象。
 * @param {object | null} parentServer - 如果是添加子服，传入父服对象
 * @returns {object} 一个新的服务器数据对象。
 */
function createBlankServer(parentServer: Server | null = null): Partial<Server> {
  return {
    ip: "",
    comment: "",
    tag: "",
    tag_color: "FF9800",
    tag_color_with_hash: "#FF9800",
    server_type: SERVER_TYPE.STANDALONE, // 默认为 standalone
    parent_ip: parentServer ? parentServer.ip : "", // (v16) 预设 parent_ip
    selectedPreset: "",
    ignore_in_list: false,
    children: [] // 确保 children 存在
  }
}

/**
 * @description (v16) 打开服务器弹窗（添加或编辑模式）
 * @param {object | null} [serverToEdit=null] - 要编辑的服务器对象。如果为 null，则为添加模式。
 * @param {object | null} [parentServer=null] - (仅添加模式) 如果要添加子服，传入父服。
 */
async function openServerModal(serverToEdit: Server | null = null, parentServer: Server | null = null) {
  // 1. (竞争条件修复) 清除任何待处理的关闭超时
  if (cancelModalTimeout) {
    clearTimeout(cancelModalTimeout)
    cancelModalTimeout = null
  }

  isSaving.value = false; // 重置保存状态

  // 2. 先清空数据 (确保 v-if 触发)
  currentServerData.value = null

  if (serverToEdit) {
    // --- 编辑模式 ---
    modalMode.value = MODAL_MODE.EDIT
    // 存储原始 IP 用于验证
    editingServerIp.value = serverToEdit.ip
    // 填充数据为 *深拷贝*
    await nextTick() // 等待 v-if=null 生效
    currentServerData.value = JSON.parse(JSON.stringify(serverToEdit))
  } else {
    // --- 添加模式 ---
    modalMode.value = MODAL_MODE.ADD
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
  cancelModalTimeout = window.setTimeout(() => {
    currentServerData.value = null
    editingServerIp.value = null
    cancelModalTimeout = null
  }, 300)
}

/**
 * @description (优化) 检查设置父服务器时是否会造成循环依赖。
 * @param {string} serverIp - 当前服务器的 IP。
 * @param {string} newParentIp - 计划设置的父服务器 IP。
 * @returns {boolean} - 如果会造成循环则返回 true。
 */
function checkForCircularDependency(serverIp: string, newParentIp: string): boolean {
  if (!newParentIp || serverIp === newParentIp) {
    return false; // 没有父级或父级是自己，不算循环 (由其他验证处理)
  }

  let currentIp: string | undefined = newParentIp;
  const visited = new Set<string>([serverIp]); // 将当前节点加入访问集合

  while (currentIp) {
    if (visited.has(currentIp)) {
      return true; // 发现循环
    }
    visited.add(currentIp);

    const parentServer = config.value.servers.find(s => s.ip === currentIp);
    currentIp = parentServer ? parentServer.parent_ip : undefined;
  }

  return false;
}


/**
 * @description (v16) 保存“添加”或“编辑”的服务器 (已重构)
 */
async function saveServer() {
  if (!currentServerData.value || isSaving.value) return;

  isSaving.value = true;
  const server = currentServerData.value as Server;
  const newIp = (server.ip || "").trim();

  // --- 1. 验证 ---
  if (newIp === '') {
    showAlert('服务器地址 (IP) 不能为空！', '保存失败');
    isSaving.value = false;
    return;
  }

  const isIpChanged = newIp !== editingServerIp.value;
  if (isIpChanged) {
    const isDuplicate = config.value.servers.some(s => s.ip === newIp);
    if (isDuplicate) {
      showAlert(`服务器地址 (IP) "${newIp}" 已存在！\n请使用唯一的 IP 地址。`, '保存失败');
      isSaving.value = false;
      return;
    }
  }

  if (server.parent_ip) {
    // 验证1: 父服务器不能成为子服务器
    if (modalMode.value === MODAL_MODE.EDIT && server.children && server.children.length > 0) {
      showAlert(
        `服务器 [${server.ip}] 本身是一个父服务器，因此不能被设置为其他服务器的子服务器。`,
        '保存失败'
      );
      isSaving.value = false;
      return;
    }
    // 验证2: (优化) 防止循环依赖
    if (checkForCircularDependency(editingServerIp.value || newIp, server.parent_ip)) {
      showAlert(
        `无法将服务器 [${server.parent_ip}] 设置为父服务器，因为这会创建一个循环依赖关系。`,
        '保存失败'
      );
      isSaving.value = false;
      return;
    }
  }

  // --- 2. 更新数据 ---
  server.ip = newIp;

  // 根据 parent_ip 自动设置 server_type
  if (server.parent_ip) {
    server.server_type = SERVER_TYPE.CHILD;
    // 自动将独立的父服转换为 'parent' 类型
    const parent = config.value.servers.find(s => s.ip === server.parent_ip);
    if (parent && parent.server_type === SERVER_TYPE.STANDALONE) {
      parent.server_type = SERVER_TYPE.PARENT;
    }
  } else {
    // 如果它有子节点，将在 flattenTreeAndSync 中被更正为 'parent'
    server.server_type = SERVER_TYPE.STANDALONE;
  }

  // --- 3. 应用更改到扁平列表 ---
  if (modalMode.value === MODAL_MODE.EDIT) {
    // 如果 IP 地址被修改，需要更新所有子服务器的 parent_ip
    if (isIpChanged && editingServerIp.value) {
      config.value.servers.forEach(s => {
        if (s.parent_ip === editingServerIp.value) {
          s.parent_ip = newIp;
        }
      });
    }

    // 找到并更新原始服务器对象
    const originalServer = config.value.servers.find(s => s.ip === editingServerIp.value);
    if (originalServer) {
      Object.assign(originalServer, server);
    }
  } else {
    // 添加新服务器
    config.value.servers.push(server as Server);
  }

  // --- 4. 收尾 ---
  // 延迟以观察加载状态
  await new Promise(resolve => setTimeout(resolve, 200));
  isSaving.value = false;
  closeServerModal();
  showToast(modalMode.value === MODAL_MODE.ADD ? '服务器已成功添加！' : '服务器已成功更新！');
}

/**
 * @description 切换父服务器下拉框的显示
 */
function toggleParentSelect() {
  isParentSelectOpen.value = !isParentSelectOpen.value
  if (isParentSelectOpen.value) {
    // 每次打开时重置高亮选项
    const currentIndex = potentialParentServers.value.findIndex(p => p.ip === currentServerData.value?.parent_ip);
    activeParentIndex.value = currentIndex > -1 ? currentIndex + 1 : 0; // +1 因为第一个是“无”
  } else {
    activeParentIndex.value = -1;
  }
}

/**
 * @description (优化) 处理父服务器下拉框的键盘导航
 */
function handleParentSelectKeydown(event: KeyboardEvent) {
  if (!isParentSelectOpen.value) return;

  const optionsCount = potentialParentServers.value.length + 1; // +1 for the "none" option
  if (event.key === 'ArrowDown') {
    event.preventDefault();
    activeParentIndex.value = (activeParentIndex.value + 1) % optionsCount;
  } else if (event.key === 'ArrowUp') {
    event.preventDefault();
    activeParentIndex.value = (activeParentIndex.value - 1 + optionsCount) % optionsCount;
  } else if (event.key === 'Enter') {
    event.preventDefault();
    if (activeParentIndex.value >= 0) {
      const selectedIp = activeParentIndex.value === 0
        ? ''
        : (potentialParentServers.value[activeParentIndex.value - 1]?.ip ?? '');
      if (selectedIp !== editingServerIp.value) { // 确保不会将自己设为父级
        selectParent(selectedIp);
      }
    }
  }
  // 确保高亮选项在可视区域内
  nextTick(() => {
    const highlightedElement = parentSelectOptionsRef.value?.querySelector('.is-active');
    highlightedElement?.scrollIntoView({ block: 'nearest' });
  });
}

/**
 * @description 选择一个新的父服务器
 */
function selectParent(parentIp: string) {
  if (currentServerData.value) {
    currentServerData.value.parent_ip = parentIp
  }
  isParentSelectOpen.value = false
    // 将焦点返回给触发器按钮
    ; (parentSelectTriggerRef.value as HTMLElement | null)?.focus()
}

/**
 * @description (v17) 处理点击外部以关闭自定义下拉框
 */
function handleClickOutsideParentSelect(event: MouseEvent) {
  if (
    isParentSelectOpen.value &&
    parentSelectTriggerRef.value && !(parentSelectTriggerRef.value as HTMLElement).contains(event.target as Node) &&
    parentSelectOptionsRef.value && !(parentSelectOptionsRef.value as HTMLElement).contains(event.target as Node)
  ) {
    isParentSelectOpen.value = false
  }
}

/**
 * @description 切换快捷预设下拉框的显示
 */
function togglePresetSelect() {
  isPresetSelectOpen.value = !isPresetSelectOpen.value;
  if (isPresetSelectOpen.value) {
    const presetKeys = Object.keys(presets);
    const currentIndex = presetKeys.indexOf(currentServerData.value?.selectedPreset || '');
    activePresetIndex.value = currentIndex > -1 ? currentIndex + 1 : 0; // +1 for "custom"
  } else {
    activePresetIndex.value = -1;
  }
}

/**
 * @description (优化) 处理预设下拉框的键盘导航
 */
function handlePresetSelectKeydown(event: KeyboardEvent) {
  if (!isPresetSelectOpen.value) return;

  const optionsCount = Object.keys(presets).length + 1; // +1 for "custom"
  if (event.key === 'ArrowDown') {
    event.preventDefault();
    activePresetIndex.value = (activePresetIndex.value + 1) % optionsCount;
  } else if (event.key === 'ArrowUp') {
    event.preventDefault();
    activePresetIndex.value = (activePresetIndex.value - 1 + optionsCount) % optionsCount;
  } else if (event.key === 'Enter') {
    event.preventDefault();
    if (activePresetIndex.value >= 0) {
      const presetKeys = Object.keys(presets);
      const selectedKey = activePresetIndex.value === 0 ? '' : presetKeys[activePresetIndex.value - 1];
      selectPreset(selectedKey as string);
    }
  }
  // 确保高亮选项在可视区域内
  nextTick(() => {
    const highlightedElement = (presetSelectOptionsRef.value as HTMLElement | null)?.querySelector('.is-active');
    (highlightedElement as HTMLElement | null)?.scrollIntoView({ block: 'nearest' });
  });
}

/**
 * @description 选择一个新的预设
 */
function selectPreset(presetKey: string) {
  if (currentServerData.value) {
    currentServerData.value.selectedPreset = presetKey
    // 关键: 选择后要立即应用 (applyPreset 会处理空 key)
    applyPreset(currentServerData.value as Server);
  }
  isPresetSelectOpen.value = false
  activePresetIndex.value = -1; // 重置
  ; (presetSelectTriggerRef.value as HTMLElement | null)?.focus()
}

/**
 * @description (v17.2) 处理点击外部以关闭快捷预设下拉框
 */
function handleClickOutsidePresetSelect(event: MouseEvent) {
  if (
    isPresetSelectOpen.value &&
    presetSelectTriggerRef.value && !(presetSelectTriggerRef.value as HTMLElement).contains(event.target as Node) &&
    presetSelectOptionsRef.value && !(presetSelectOptionsRef.value as HTMLElement).contains(event.target as Node)
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
    ; (colorPickerTriggerRef.value as HTMLElement | null)?.focus() // 焦点返回
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

function getContrastColor(hexColor: string): string {
  if (!hexColor || hexColor.length < 7) return '#000000';
  const r = parseInt(hexColor.substr(1, 2), 16);
  const g = parseInt(hexColor.substr(3, 2), 16);
  const b = parseInt(hexColor.substr(5, 2), 16);
  const yiq = ((r * 299) + (g * 587) + (b * 114)) / 1000;
  return (yiq >= 128) ? '#000000' : '#FFFFFF';
}

function sanitizeIpForId(ip: string | undefined): string {
  if (!ip) return 'new-server';
  return ip.replace(/[^a-zA-Z0-9_-]/g, '_');
}

/**
 * @description (v14) 核心重构：将扁平的服务器列表转换为嵌套树形结构。
 */
function buildTree(flatList: Server[]): Server[] {
  const map: Record<string, Server> = {}
  const serversWithChildren: Server[] = flatList
    .map(server => {
      const serverCopy = { ...server, children: [] }
      map[serverCopy.ip] = serverCopy
      return serverCopy
    })

  const tree: Server[] = []

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
 * @description (v14) 拖拽结束后，重建 `config.value.servers`。
 */
function flattenTreeAndSync() {
  const newFlatList: Server[] = []
  let priorityCounter = 0

  function traverse(nodes: Server[], parentIp = "") {
    if (!nodes) return

    nodes.forEach((server) => {
      server.parent_ip = parentIp

      if (parentIp) {
        server.server_type = 'child'
      }
      else if (server.children && server.children.length > 0) {
        server.server_type = 'parent'
      } else {
        server.server_type = 'standalone'
      }

      server.priority = (priorityCounter + 1) * 10
      priorityCounter++

      const { children, ...flatServer } = server
      newFlatList.push(flatServer as Server)

      if (children) {
        traverse(children, server.ip)
      }
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
 * @description “加载配置”按钮的点击事件处理器。
 */
async function loadConfig() {
  // 1. 确认覆盖
  if (config.value.servers.length > 0) {
    const confirmed = await showConfirm(
      '您确定要加载新的配置吗？\n当前编辑器中的所有服务器都将被替换。',
      '覆盖确认'
    );
    if (!confirmed) {
      return;
    }
  }

  const input = jsonInput.value.trim();
  if (!input) {
    showAlert('输入框不能为空。', '提示');
    return;
  }

  let configToParse: string | AppConfig | null = null;

  // 2. 检查输入类型 (URL vs JSON)
  if (input.startsWith('http') && input.includes('?data=')) {
    try {
      const url = new URL(input);
      const dataFromUrl = url.searchParams.get('data');
      if (dataFromUrl) {
        const decompressed = decompressConfig(dataFromUrl);
        if (decompressed) {
          configToParse = decompressed;
        } else {
          showAlert('无法解压来自链接的数据，链接可能不完整或已损坏。', '导入失败');
          return;
        }
      } else {
        // 是一个 URL 但没有 data 参数，这很奇怪，当作错误处理
        showAlert('链接中未找到有效的“data”参数。', '导入失败');
        return;
      }
    } catch (e) {
      showAlert('输入的分享链接无效。', '导入失败');
      return;
    }
  } else {
    // 3. 假定为 JSON 字符串
    configToParse = input;
  }

  // 4. 解析和设置配置
  if (configToParse) {
    const success = parseAndSetConfig(configToParse);
    if (success) {
      showToast(`配置已成功加载！共导入 ${config.value.servers.length} 个服务器。`);
      jsonInput.value = ''; // 成功后清空
    }
    // `parseAndSetConfig` 内部会在失败时显示 alert
  }
}

/**
 * @description 解析 JSON 字符串并设置到 `config` 状态中。(已重构)
 * @returns {boolean} - 解析和加载是否成功。
 */

function parseAndSetConfig(input: string | AppConfig): boolean {
  // (优化) 将内部逻辑拆分为独立的、可测试的函数
  function flattenImportedServers(servers: any[], parentIp = ""): any[] {
    if (!Array.isArray(servers)) return [];
    const flatList: any[] = [];
    servers.forEach(s => {
      s.parent_ip = s.parent_ip || parentIp;
      const children = s.children;
      delete s.children;
      flatList.push(s);
      if (children) {
        flatList.push(...flattenImportedServers(children, s.ip));
      }
    });
    return flatList;
  }

  function processAndValidate(flatList: any[]): Server[] {
    const ipSet = new Set<string>();
    const duplicates: string[] = [];
    const validatedServers: Server[] = [];

    flatList.forEach(s => {
      if (!s.ip) {
        throw new Error('导入失败：存在没有 IP 地址的服务器条目。');
      }
      if (ipSet.has(s.ip)) {
        duplicates.push(s.ip);
      }
      ipSet.add(s.ip);

      // 补全 UI 辅助属性和默认值
      const server: Partial<Server> = {
        ...s,
        tag_color_with_hash: (s.tag_color && s.tag_color.length > 0) ? '#' + s.tag_color : '#FF9800',
        selectedPreset: "",
        ignore_in_list: s.ignore_in_list || false,
        comment: s.comment || "",
        children: []
      };
      validatedServers.push(server as Server);
    });

    if (duplicates.length > 0) {
      throw new Error(`导入失败：JSON 数据中包含重复的 IP 地址。\n重复项: ${[...new Set(duplicates)].join(', ')}`);
    }

    // 按 priority 排序
    return validatedServers.sort((a, b) => (a.priority || 0) - (b.priority || 0));
  }

  let importedConfig: any;
  try {
    if (typeof input === 'string') {
      importedConfig = JSON.parse(input);
    } else {
      importedConfig = input;
    }

    const flatServers = flattenImportedServers(importedConfig.servers || []);
    const validatedServers = processAndValidate(flatServers);

    let footerContent = importedConfig.footer;
    if (typeof footerContent === 'object' && footerContent !== null && 'text' in footerContent) {
      footerContent = (footerContent as { text: string }).text;
    }

    config.value.servers = validatedServers;
    config.value.footer = footerContent || "";
    config.value.show_offline_by_default = importedConfig.show_offline_by_default || false;

    return true;
  } catch (e) {
    showAlert(e instanceof Error ? e.message : String(e), '导入失败');
    return false;
  }
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
  // (优化) 优先从 URL 参数导入
  const urlParams = new URLSearchParams(window.location.search);
  const dataFromUrl = urlParams.get('data');

  if (dataFromUrl) {
    const decompressedConfig = decompressConfig(dataFromUrl);
    if (decompressedConfig) {
      const success = parseAndSetConfig(decompressedConfig);
      if (success) {
        showToast('已从 URL 成功导入配置！');
        // 清理 URL，避免刷新时重复导入
        window.history.replaceState({}, document.title, window.location.pathname);
      } else {
        showAlert('从 URL 导入配置失败，数据可能已损坏。', '导入失败');
      }
    } else {
      showAlert('无法解压来自 URL 的数据，链接可能不完整或已损坏。', '导入失败');
    }
    return; // 处理完 URL 后即返回，不再处理剪贴板
  }

  // 如果 URL 中没有数据，再尝试从剪贴板导入
  try {
    const text = await navigator.clipboard.readText();
    if (!text) return; // 剪贴板为空

    const data = JSON.parse(text);

    if (data && (Array.isArray(data.servers) || data.hasOwnProperty('footer'))) {
      jsonInput.value = text;
      parseAndSetConfig(text);
      showToast('已从剪贴板自动导入配置。');
    }
  } catch (e: any) {
    console.warn('Failed to auto-import from clipboard:', e.message);
  }

  if (config.value.servers.length === 0 && !config.value.footer) {
    parseAndSetConfig(`{}`);
  }

  const savedTheme = localStorage.getItem('theme')
  if (savedTheme) {
    applyTheme(savedTheme === 'dark')
  } else {
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
          <h3>全局设置</h3>
          <div class="form-group">
            <label for="global-footer"
              style="font-weight: normal; font-size: 0.95rem; color: var(--color-text-primary);">
              页脚文本 (Footer)
            </label>
            <input id="global-footer" type="text" v-model="config.footer" placeholder="输入页脚文本" />
          </div>

          <div class="form-group-toggle">
            <label for="global_show_offline" class="toggle-switch-label">
              默认显示离线服务器
            </label>
            <label class="toggle-switch">
              <input type="checkbox" v-model="config.show_offline_by_default" id="global_show_offline"
                class="toggle-switch-input" />
              <span class="toggle-switch-slider"></span>
            </label>
            <p class="form-help-text" style="margin-left: 10px; margin-top: 0; margin-bottom: 0;">
              (勾选后, /mcs 命令将默认显示所有服务器, 相当于 /mcs all)
            </p>
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

          <draggable v-if="serverTree && serverTree.length > 0" v-model="serverTree" :item-key="(server: Server) => server.ip" handle=".drag-handle"
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
                    <div class="simple-info-line">
                      <span v-if="server.tag" class="simple-tag" :style="{
                        backgroundColor: server.tag_color_with_hash,
                        color: getContrastColor(server.tag_color_with_hash)
                      }">
                        {{ server.tag }}
                      </span>
                      <span class="simple-comment" v-if="server.comment">{{ server.comment }}</span>
                    </div>
                    <div class="simple-info-line">
                      <span class="simple-ip" :class="{ 'with-comment': server.comment }">
                        {{ server.comment ? '(' + server.ip + ')' : server.ip }}
                      </span>
                      <span v-if="server.ignore_in_list" class="simple-ignored-badge">(已隐藏)</span>
                    </div>
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

                <draggable v-model="server.children" :item-key="(child: Server) => child.ip" handle=".drag-handle"
                  :group="{ name: 'servers', pull: true, put: true }" @end="flattenTreeAndSync"
                  class="server-list child-list" :name="'server-list-anim-child'">
                  <template #item="{ element: childServer }">
                    <div :key="childServer.ip" class="server-item-simple is-child" :class="{
                      'is-ignored': childServer.ignore_in_list
                    }">
                      <div class="drag-handle">⠿</div>
                      <div class="simple-info">
                        <div class="simple-info-line">
                          <span v-if="childServer.tag" class="simple-tag" :style="{
                            backgroundColor: childServer.tag_color_with_hash,
                            color: getContrastColor(childServer.tag_color_with_hash)
                          }">
                            {{ childServer.tag }}
                          </span>
                          <span class="simple-comment" v-if="childServer.comment">{{ childServer.comment }}</span>
                        </div>
                        <div class="simple-info-line">
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

          <div v-else class="empty-state">
            <h3>尚未添加服务器</h3>
            <p>点击上方"添加"按钮开始配置，或使用导入功能粘贴数据</p>
          </div>
        </div>
      </div>
    </div>

    <div class="panel io-panel">
      <div class="panel-body">
        <div class="form-section">
          <h3 class="io-header">1. 导入 (Import)</h3>
          <p>粘贴分享链接或从机器人 /mcs export 导出的 JSON：</p>
          <div class="form-group">
            <textarea v-model="jsonInput" rows="8" placeholder="在此粘贴分享链接或 JSON..."></textarea>
          </div>
          <button @click="loadConfig" class="btn btn-primary">
            <font-awesome-icon :icon="faUpload" /> 加载配置
          </button>
        </div>

        <div class="form-section">
          <h3 class="io-header">2. 导出 (Export)</h3>
          <p>复制生成的命令，并将其发送到QQ群聊中：</p>
          <div class="form-group">
            <textarea :value="importCommand" rows="8" readonly></textarea>
          </div>
          <button @click="copyImportCommand" class="btn btn-secondary">
            <font-awesome-icon :icon="faClipboard" /> 复制导入命令
          </button>
          <button @click="downloadJson" class="btn btn-primary" style="margin-left: 10px;">
            <font-awesome-icon :icon="faSave" /> 导出JSON
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
                    <input type="text" v-model="currentServerData.tag"
                      @input="checkIfCustom(currentServerData as Server)" placeholder="留空则不显示"
                      class="form-compound-input-text" />

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
                              @pureColorChange="onColorInput(currentServerData as Server)" />
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
                      @click="togglePresetSelect" @keydown="handlePresetSelectKeydown" aria-haspopup="listbox"
                      :aria-expanded="isPresetSelectOpen"
                      :aria-activedescendant="(isPresetSelectOpen && activePresetIndex > -1 ? `preset-option-${activePresetIndex}` : undefined)">

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

                        <li :id="`preset-option-0`" class="custom-select-option"
                          :class="{ 'is-selected': !currentServerData.selectedPreset, 'is-active': activePresetIndex === 0 }"
                          @click="selectPreset('')" role="option" :aria-selected="!currentServerData.selectedPreset">
                          <span class="option-placeholder">-- 自定义 --</span>
                        </li>

                        <li v-for="(preset, key, index) in presets" :key="key" :id="`preset-option-${index + 1}`"
                          class="custom-select-option" :class="{
                            'is-selected': currentServerData.selectedPreset === key,
                            'is-active': activePresetIndex === index + 1
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
                      @click="toggleParentSelect" @keydown="handleParentSelectKeydown" :disabled="(modalMode === MODAL_MODE.EDIT && currentServerData.children && currentServerData.children.length > 0) ||
                        (potentialParentServers.length === 0)
                        " aria-haspopup="listbox" :aria-expanded="isParentSelectOpen"
                      :aria-activedescendant="(isParentSelectOpen && activeParentIndex > -1 ? `parent-option-${activeParentIndex}` : undefined)">

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

                        <li :id="`parent-option-0`" class="custom-select-option"
                          :class="{ 'is-selected': !currentServerData.parent_ip, 'is-active': activeParentIndex === 0 }"
                          @click="selectParent('')" role="option" :aria-selected="!currentServerData.parent_ip">
                          <span class="option-placeholder">-- 默认为根服务器 --</span>
                        </li>

                        <li v-for="(parent, index) in potentialParentServers" :key="parent.ip"
                          :id="`parent-option-${index + 1}`" class="custom-select-option" :class="{
                            'is-selected': currentServerData.parent_ip === parent.ip,
                            'is-disabled': parent.ip === editingServerIp,
                            'is-active': activeParentIndex === index + 1
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
                <div style="padding-top: 10px;">
                  <div class="form-group-toggle" style="margin-top: 0;">
                    <label class="toggle-switch">
                      <input type="checkbox" v-model="currentServerData.ignore_in_list"
                        :id="'ignore_mod_' + sanitizeIpForId(currentServerData.ip)" class="toggle-switch-input" />
                      <span class="toggle-switch-slider"></span>
                    </label>
                    <label :for="'ignore_mod_' + sanitizeIpForId(currentServerData.ip || 'new')"
                      class="toggle-switch-label">
                      <font-awesome-icon :icon="faEyeSlash" /> 在列表中隐藏
                    </label>
                  </div>
                  <p class="form-help-text" style="margin-top: 8px; margin-bottom: 0; padding-left: 5px;">
                    (勾选后, 该服务器将不会显示在 /mcs 的列表图片中)
                  </p>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button @click="closeServerModal" class="btn-modal-cancel" type="button">
              <font-awesome-icon :icon="faTimes" /> 取消
            </button>
            <button @click="saveServer" class="btn-modal-confirm" type="button" :disabled="isSaving">
              <font-awesome-icon :icon="isSaving ? faSpinner : faSave" :spin="isSaving" />
              <span v-if="isSaving">正在保存...</span>
              <span v-else>{{ modalMode === MODAL_MODE.ADD ? '确认添加' : '保存更改' }}</span>
            </button>
          </div>

        </div>
      </div>
    </transition>

    <!-- (优化) Toast 小提示 -->
    <transition name="toast-fade">
      <div v-if="isToastVisible" class="toast-notification">
        {{ toastMessage }}
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
}

.header-actions {
  display: flex;
  gap: 10px;
}

.server-list {
  min-height: 50px;
  margin-top: 15px;
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

/* --- (已修复) 已隐藏服务器的删除线 (方案3) --- */

/* (重要) 移除父容器的删除线，避免继承问题 */
.server-item-simple.is-ignored .simple-info {
  text-decoration: none;
}

/* 只对需要删除线的文本元素单独应用 */
.server-item-simple.is-ignored .simple-tag,
.server-item-simple.is-ignored .simple-comment,
.server-item-simple.is-ignored .simple-ip {
  text-decoration: line-through;
  opacity: 0.7;
  /* 增加隐藏效果 */
}

/* 确保徽章始终清晰，没有删除线 */
.server-item-simple.is-ignored .simple-ignored-badge {
  text-decoration: none;
  opacity: 1;
  /* 确保徽章清晰 */
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

.simple-info-line {
  display: contents;
  /* (v17.33) 核心: 桌面端让此包装器"消失" */
}

.simple-tag {
  font-size: 0.85rem;
  font-weight: 500;
  padding: 3px 8px;
  border-radius: 4px;
  flex-shrink: 0;
  /* <--- (v17.24) 新增此行 */
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
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex-shrink: 1;
  /* 允许 IP 地址收缩 */
  min-width: 0;
  /* 允许收缩到 0 */
  /* 已移除 flex-basis 和 flex-grow，使其在宽屏下正常排列 */
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
  flex-shrink: 0;
  /* <--- (v17.24) 新增此行 */
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
  padding: 20px;
  color: var(--color-text-secondary);
  /* (修正) */
  background: var(--color-surface-muted);
  /* (修正) */
  border-radius: 8px;
  margin-top: 15px;
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
  background-color: var(--color-body-gradient-start);
}

.custom-select-option.is-active {
  background-color: var(--color-body-gradient-start);
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
}

/* (优化) Toast 小提示样式 */
.toast-notification {
  position: fixed;
  top: 80px;
  /* 调整位置，使其更显眼 */
  left: 50%;
  transform: translateX(-50%);
  background-color: var(--color-success);
  color: #fff;
  padding: 10px 20px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 2000;
  font-weight: 500;
}

.toast-fade-enter-active,
.toast-fade-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}

.toast-fade-enter-from,
.toast-fade-leave-to {
  opacity: 0;
  transform: translate(-50%, -20px);
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
.toggle-switch-label svg,
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

/* --- (v17.35) 新增：切换开关样式 --- */
.form-group-toggle {
  display: flex;
  align-items: center;
  /* 关键：修复对齐问题 */
  gap: 12px;
  margin-top: 15px;
  padding-left: 5px;
}

.toggle-switch-label {
  color: var(--color-text-primary);
  font-weight: 500;
  cursor: pointer;
  user-select: none;
  font-size: 0.95rem;
  /* 匹配 .form-group-checkbox label */
}

.toggle-switch {
  position: relative;
  display: inline-block;
  width: 50px;
  /* 开关宽度 */
  height: 28px;
  /* 开关高度 */
  flex-shrink: 0;
  /* 防止被压缩 */
}

/* 隐藏默认的 checkbox */
.toggle-switch-input {
  opacity: 0;
  width: 0;
  height: 0;
}

/* 开关的 "轨道" */
.toggle-switch-slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: var(--color-border);
  transition: .3s;
  border-radius: 28px;
  /* 圆角 */
}

/* 开关的 "滑块" (白色圆点) */
.toggle-switch-slider:before {
  position: absolute;
  content: "";
  height: 20px;
  width: 20px;
  left: 4px;
  bottom: 4px;
  background-color: white;
  transition: .3s;
  border-radius: 50%;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  /* 轻微阴影 */
}

/* 选中时的样式：轨道变色 */
.toggle-switch-input:checked+.toggle-switch-slider {
  background-color: var(--color-primary);
}

/* 选中时的样式：滑块移动 */
.toggle-switch-input:checked+.toggle-switch-slider:before {
  transform: translateX(22px);
  /* 移动距离: 50(宽) - 20(滑块) - 4(左距) - 4(右距) = 22px */
}

/* 焦点样式 (用于键盘导航) */
.toggle-switch-input:focus-visible+.toggle-switch-slider {
  box-shadow: 0 0 0 3px var(--color-focus-outline);
}

/* --- (v17.35) 切换开关样式结束 --- */

/* --- (v17.22) 移动端适配：使用 HTML 包装器 --- */

@media (max-width: 600px) {

  /* 1. (不变) 父容器顶部对齐 */
  .server-item-simple {
    padding: 10px 10px 12px 10px;
    align-items: flex-start;
  }

  /* 2. (v17.33) 重构 simple-info 为垂直布局 */
  .simple-info {
    flex-direction: column;
    /* 强制垂直堆叠 */
    align-items: flex-start;
    /* 左对齐 */
    gap: 4px;
    /* 行间距 */
    white-space: normal;
    /* 允许换行（虽然我们内部会阻止）*/

    /* (v17.33) 移除旧规则 */
    /* flex-wrap: wrap; */
    /* align-items: center; */
  }

  /* (v17.33) 新增：行容器样式 */
  .simple-info-line {
    display: flex;
    /* 变为 flex 布局 */
    align-items: center;
    /* 内部元素垂直居中 */
    gap: 8px;
    /* 元素间距 (tag 和 comment 之间) */
    width: 100%;
    /* 占满父容器宽度 */
  }

  /* --- 4. 第一行项目 --- */
  .simple-tag {
    white-space: nowrap;
    flex-shrink: 0;
  }

  /* (v17.33) 重构 .simple-comment */
  .simple-comment {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    flex-shrink: 1;
    min-width: 0;
    flex: 1;
    /* 核心：让 comment 填满第一行的剩余空间 */
  }

  /* --- 5. 第二行项目 --- */
  /* (v17.33) 重构 .simple-ip */
  .simple-ip {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    flex-shrink: 1;
    min-width: 0;
    /* 已移除 flex: 1，使其不再推开徽章 */
  }

  .simple-ignored-badge {
    white-space: nowrap;
    flex-shrink: 0;
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

  .simple-ip.with-comment {
    margin-left: 0;
    font-size: 0.85rem;
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
