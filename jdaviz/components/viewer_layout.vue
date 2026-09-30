<template>
  <div ref="host" class="jdz-viewer-layout jdz-native-layout" @pointermove="move"
       @pointercancel="cancel" @lostpointercapture="lostCapture">
    <section v-for="pane in geometry.panes" :key="pane.key" class="jdz-native-pane"
             :data-pane="pane.key" :style="boxStyle(pane)">
      <header v-if="hasHeaders" class="jdz-native-header">
        <div :ref="(element) => setTabElement(pane.key, element)" class="jdz-native-tabs" role="tablist" aria-label="Viewers">
          <template v-for="(id, index) in pane.node.viewers" :key="id">
            <div v-if="drop?.tab?.paneKey === pane.key && drop.tab.index === index"
                 class="jdz-native-tab-placeholder" aria-hidden="true"></div>
            <div class="jdz-native-tab"
                 :class="{ active: id === pane.node.activeViewerId }">
              <button type="button" class="jdz-native-tab-label" role="tab"
                      :data-viewer-tab-id="id" :aria-selected="id === pane.node.activeViewerId"
                      :tabindex="id === pane.node.activeViewerId ? 0 : -1" :title="title(id)"
                      @click="select(id)" @keydown="tabKey($event, pane.node, id)"
                      @pointerdown="start($event, { id })" @auxclick.middle.prevent="close(id)">
                <span>{{ title(id) }}</span>
              </button>
              <button v-if="metadata.get(id)?.closable !== false" type="button"
                      :aria-label="`Close ${title(id)}`" @pointerdown.stop @click.stop="close(id)">×</button>
            </div>
          </template>
          <div v-if="drop?.tab?.paneKey === pane.key && drop.tab.index === pane.node.viewers.length"
               class="jdz-native-tab-placeholder" aria-hidden="true"></div>
        </div>
        <select v-if="overflowPanes.has(pane.key)" class="jdz-native-overflow"
                aria-label="Select viewer tab" title="Select viewer tab" :value="pane.node.activeViewerId"
                @pointerdown.stop @change="select($event.target.value)">
          <option v-for="id in pane.node.viewers" :key="id" :value="id">{{ title(id) }}</option>
        </select>
        <button type="button" class="jdz-native-maximize"
                :aria-label="current.maximizedViewerId ? 'Restore viewer' : 'Maximize viewer'"
                @click="maximize(pane.node.activeViewerId)">{{ current.maximizedViewerId ? '▣' : '□' }}</button>
      </header>
    </section>
    <div v-for="(splitter, index) in geometry.splitters" :key="index" class="jdz-native-splitter"
         :class="`jdz-native-${splitter.axis}`" :style="boxStyle(splitter)" role="separator" tabindex="0"
         :aria-orientation="splitter.axis === 'row' ? 'vertical' : 'horizontal'"
         v-bind="splitterAria(splitter)"
         aria-label="Resize viewers" @pointerdown="start($event, { splitter })"
         @keydown="splitKey($event, splitter)"></div>
    <div class="jdz-viewer-layout__viewers">
      <div v-for="viewer in viewers" :key="viewer.id"
           :ref="(element) => setViewerElement(viewer.id, element)"
           class="jdz-viewer-layout__viewer" :data-viewer-id="viewer.id">
        <slot name="viewer" :viewer-id="viewer.id" :viewer="viewer">
          <jupyter-widget :widget="viewer.widget" :key="viewer.widget || viewer.id"></jupyter-widget>
        </slot>
      </div>
    </div>
    <div v-if="gesture?.active" class="jdz-native-drag-shield"></div>
    <div v-if="gesture?.active && !gesture.splitter" class="jdz-native-ghost"
         :style="boxStyle(ghost)" aria-hidden="true">
      <div class="jdz-native-ghost-title">{{ title(gesture.id) }}</div>
    </div>
    <div v-if="drop" class="jdz-native-drop" :class="{ 'jdz-native-drop-tab': drop.tab }"
         :style="boxStyle(drop.preview)"></div>
  </div>
</template>

<script>
const GAP = 5
const HEADER = 28
const TAB_DROP_WIDTH = 100
const MIN_PANE_SIZE = 50
const cloneLayout = (layout) => JSON.parse(JSON.stringify(layout))
const sum = (values) => values.reduce((a, b) => a + b, 0)

function findStack(node, id, parent = null) {
  if (!node) return null
  if (node.type === 'stack') return node.viewers.includes(id) ? { node, parent } : null
  for (const child of node.children) {
    const found = findStack(child, id, node)
    if (found) return found
  }
  return null
}

// The Python boundary validates input. Edits only need pruning, defaults, and
// the same positive twelve-digit weight normalization as the canonical model.
export function normalizeLayout(layout) {
  function prune(node) {
    if (!node) return null
    if (node.type === 'stack') {
      if (!node.viewers.length) return null
      if (!node.viewers.includes(node.activeViewerId)) node.activeViewerId = node.viewers[0]
    } else {
      node.children = node.children.map(prune).filter(Boolean)
      if (!node.children.length) return null
      if (node.children.length === 1) {
        const child = node.children[0]
        child.weight = node.weight
        return child
      }
      const weights = node.children.map((child) => child.weight ?? 1)
      const scale = Math.max(...weights)
      const scaled = weights.map((weight) => weight / scale)
      const total = sum(scaled)
      const normalized = scaled.map((weight) => Math.max(+(weight / total).toFixed(12), 1e-12))
      const largest = normalized.indexOf(Math.max(...normalized))
      normalized[largest] = +(normalized[largest] + (1 - sum(normalized))).toFixed(12)
      node.children.forEach((child, i) => { child.weight = normalized[i] })
    }
    // Keep field order stable for comparisons with Python's normalized replies.
    return node.type === 'stack'
      ? { type: node.type, weight: node.weight, viewers: node.viewers, activeViewerId: node.activeViewerId }
      : { type: node.type, weight: node.weight, children: node.children }
  }
  const result = { version: layout.version, root: prune(cloneLayout(layout.root)) }
  if (result.root) delete result.root.weight
  if (layout.maximizedViewerId !== undefined) {
    const stack = findStack(result.root, layout.maximizedViewerId)
    if (stack) result.maximizedViewerId = stack.node.activeViewerId
  }
  return result
}

// Preserve selection next to the removed tab, including maximized stacks.
function removeTab(stack, id) {
  const index = stack.viewers.indexOf(id)
  if (index < 0) return
  stack.viewers.splice(index, 1)
  if (stack.activeViewerId === id) stack.activeViewerId = stack.viewers[Math.max(0, index - 1)]
}

// Edit helpers take a normalized layout and return a draft for publish().
export function closeViewer(layout, id) {
  const result = cloneLayout(layout)
  const found = findStack(result.root, id)
  if (!found) return result
  removeTab(found.node, id)
  if (result.maximizedViewerId === id) result.maximizedViewerId = found.node.activeViewerId
  return result
}

// Root-edge docking places a new pane beside the entire remaining layout.
export function dockViewerAtRoot(layout, id, side) {
  let result = cloneLayout(layout)
  const source = findStack(result.root, id)
  if (!source || !['left', 'right', 'top', 'bottom'].includes(side)) return result
  removeTab(source.node, id)
  delete result.maximizedViewerId
  // Prune the source before choosing how to split the remaining root.
  result = normalizeLayout(result)
  const stack = { type: 'stack', viewers: [id], activeViewerId: id, weight: 0.5 }
  const axis = ['left', 'right'].includes(side) ? 'row' : 'column'
  const before = ['left', 'top'].includes(side)
  if (!result.root) result.root = stack
  else if (result.root.type === axis) {
    result.root.children.forEach((child) => { child.weight /= 2 })
    result.root.children.splice(before ? 0 : result.root.children.length, 0, stack)
  } else {
    result.root.weight = 0.5
    result.root = { type: axis, children: before ? [stack, result.root] : [result.root, stack] }
  }
  return result
}

export function dockViewer(layout, id, targetId, side = 'center', index) {
  const result = cloneLayout(layout)
  const foundSource = findStack(result.root, id), foundTarget = findStack(result.root, targetId)
  if (!foundSource || !foundTarget) return result
  const source = foundSource.node
  const { node: target, parent } = foundTarget
  if (source === target && target.viewers.length === 1) return result
  const oldIndex = source.viewers.indexOf(id)
  removeTab(source, id)
  delete result.maximizedViewerId
  if (side === 'center') {
    let insertion = index ?? target.viewers.length
    if (index !== undefined && source === target && oldIndex < index) insertion--
    target.viewers.splice(insertion, 0, id)
    target.activeViewerId = id
  } else {
    const axis = ['left', 'right'].includes(side) ? 'row' : 'column'
    const before = ['left', 'top'].includes(side)
    const stack = { type: 'stack', viewers: [id], activeViewerId: id }
    if (parent?.type === axis) {
      target.weight /= 2
      stack.weight = target.weight
      parent.children.splice(parent.children.indexOf(target) + (before ? 0 : 1), 0, stack)
    } else {
      const split = { type: axis, weight: target.weight, children: before ? [stack, target] : [target, stack] }
      target.weight = stack.weight = 1
      if (parent) parent.children.splice(parent.children.indexOf(target), 1, split)
      else result.root = split
    }
  }
  return result
}

// Pixel constraints affect only this projection, never the stored weights.
export function layoutGeometry(layout, width, height) {
  const panes = [], splitters = []
  function minimum(node, axis) {
    if (node.type === 'stack') return MIN_PANE_SIZE
    const values = node.children.map((child) => minimum(child, axis))
    return node.type === axis ? sum(values) + GAP * (values.length - 1) : Math.max(...values)
  }
  function place(node, box, key) {
    if (!node) return
    if (node.type === 'stack') {
      panes.push({ ...box, node, key })
      return
    }
    const position = node.type === 'row' ? 'left' : 'top'
    const dimension = node.type === 'row' ? 'width' : 'height'
    const total = Math.max(0, box[dimension] - GAP * (node.children.length - 1))
    const minima = node.children.map((child) => minimum(child, node.type))
    let sizes = node.children.map((child) => total * child.weight)
    const deficit = sizes.reduce((sum, size, i) => sum + Math.max(0, minima[i] - size), 0)
    const slack = sizes.reduce((sum, size, i) => sum + Math.max(0, size - minima[i]), 0)
    const minTotal = sum(minima)
    if (deficit) sizes = slack >= deficit
      ? sizes.map((size, i) => Math.max(size, minima[i]) - Math.max(0, size - minima[i]) * deficit / slack)
      : minima.map((size) => total * size / minTotal)
    let offset = 0
    node.children.forEach((child, index) => {
      place(child, { ...box, [position]: box[position] + offset, [dimension]: sizes[index] }, `${key}-${index}`)
      offset += sizes[index]
      if (index < node.children.length - 1) {
        splitters.push({ ...box, [position]: box[position] + offset, [dimension]: GAP,
          axis: node.type, node, index, sizes, minima })
        offset += GAP
      }
    })
  }
  const root = (layout.maximizedViewerId && findStack(layout.root, layout.maximizedViewerId)?.node) || layout.root
  place(root, { left: 0, top: 0, width, height }, 'root')
  return { panes, splitters }
}
</script>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, shallowRef, watch } from 'vue'

const props = defineProps({
  layout: { type: Object, required: true },
  viewers: { type: Array, required: true },
  hasHeaders: { type: Boolean, default: true },
  layoutReset: { type: Number, default: 0 },
})
const emit = defineEmits(['update:layout', 'viewer-close', 'layout-applied'])
const host = ref(null)
// Hosts belong to the registry, never to a pane. Moving a tab only changes CSS.
const viewerElements = new Map()
const appliedPlacements = new WeakMap()
const tabElements = new Map()
const overflowPanes = shallowRef(new Set())
function setViewerElement(id, element) {
  if (element) {
    if (viewerElements.get(id) === element) return
    viewerElements.set(id, element)
    schedulePlacement()
  } else viewerElements.delete(id)
}
function setTabElement(key, element) {
  const old = tabElements.get(key)
  if (old === element) return
  if (old) observer?.unobserve(old)
  if (element) {
    tabElements.set(key, element)
    observer?.observe(element)
  } else tabElements.delete(key)
}
function updateOverflow() {
  const next = new Set([...tabElements].filter(([key, element]) => {
    // Measure available space without the menu itself to avoid keeping a
    // redundant menu after the pane grows just wide enough for all tabs.
    const menuWidth = overflowPanes.value.has(key) ? 28 : 0
    return element.scrollWidth > element.clientWidth + menuWidth + 1
  }).map(([key]) => key))
  if (next.size !== overflowPanes.value.size || [...next].some((key) => !overflowPanes.value.has(key))) overflowPanes.value = next
  // Horizontal scrolling keeps the active label accessible, without scrolling
  // the notebook or surrounding application.
  for (const element of tabElements.values()) {
    const tab = element.querySelector('[aria-selected="true"]')
    if (!tab) continue
    const box = tab.getBoundingClientRect(), viewport = element.getBoundingClientRect()
    if (box.left < viewport.left) element.scrollLeft -= viewport.left - box.left
    else if (box.right > viewport.right) element.scrollLeft += box.right - viewport.right
  }
}
const current = ref(normalizeLayout(props.layout))
const size = ref({ width: 0, height: 0 })
const gesture = shallowRef(null)
const drop = shallowRef(null)
const ghost = shallowRef({ left: 0, top: 0, width: 300, height: 200 })
const metadata = computed(() => new Map(props.viewers.map((viewer) => [viewer.id, viewer])))
// Detach only in the visual projection until drop; cancellation and Python
// synchronization still have the complete tree, and widget hosts stay mounted.
const displayedLayout = computed(() => {
  const edit = gesture.value
  if (!edit?.active || edit.splitter) return current.value
  const detached = closeViewer(current.value, edit.id)
  delete detached.maximizedViewerId
  return normalizeLayout(detached)
})
const geometry = computed(() => layoutGeometry(displayedLayout.value, size.value.width, size.value.height))
let observer, frame, placementQueued = false, disposed = false
const pendingLayouts = []
const title = (id) => metadata.value.get(id)?.reference || metadata.value.get(id)?.name || id
const boxStyle = (box) => Object.fromEntries(['left', 'top', 'width', 'height'].map((key) => [key, `${box[key]}px`]))
defineExpose({ snapshot: () => normalizeLayout(current.value) })

function publish(layout, viewerId) {
  const next = normalizeLayout(layout)
  const key = JSON.stringify(next)
  const changed = key !== (pendingLayouts.at(-1) ?? JSON.stringify(normalizeLayout(props.layout)))
  current.value = next
  if (viewerId || changed) pendingLayouts.push(key)
  if (viewerId) emit('viewer-close', { viewerId, layout: structuredClone(next) })
  else if (changed) emit('update:layout', structuredClone(next))
}

function select(id) {
  if (!findStack(current.value.root, id)) return
  const next = cloneLayout(current.value)
  findStack(next.root, id).node.activeViewerId = id
  publish(next)
}
function close(id) {
  const source = findStack(current.value.root, id)
  if (!source || metadata.value.get(id)?.closable === false) return
  const remaining = source.node.viewers.filter((viewer) => viewer !== id)
  const next = closeViewer(current.value, id)
  const active = remaining.length ? findStack(next.root, remaining[0]).node.activeViewerId : null
  publish(next, id)
  nextTick(() => {
    if (disposed || !host.value) return
    const tabs = [...host.value.querySelectorAll('[role=tab]')]
    const target = active
      ? tabs.find((tab) => tab.dataset.viewerTabId === active)
      : tabs.find((tab) => tab.getAttribute('aria-selected') === 'true')
    target?.focus()
  })
}
function maximize(id) {
  const next = cloneLayout(current.value)
  if (next.maximizedViewerId) delete next.maximizedViewerId
  else next.maximizedViewerId = id
  publish(next)
}
function tabKey(event, stack, id) {
  if (gesture.value) return
  let index = stack.viewers.indexOf(id)
  if (event.key === 'ArrowRight') index++
  else if (event.key === 'ArrowLeft') index--
  else if (event.key === 'Home') index = 0
  else if (event.key === 'End') index = stack.viewers.length - 1
  else if (!['Enter', ' '].includes(event.key)) return
  event.preventDefault()
  id = stack.viewers[(index + stack.viewers.length) % stack.viewers.length]
  select(id)
  nextTick(() => {
    if (disposed || !host.value) return
    const tabs = [...host.value.querySelectorAll('[role=tab]')]
    tabs.find((tab) => tab.dataset.viewerTabId === id)?.focus()
  })
}

function start(event, detail) {
  if (gesture.value || !event.isPrimary || event.button !== 0) return
  host.value.setPointerCapture(event.pointerId)
  gesture.value = { ...detail, pointerId: event.pointerId, x: event.clientX, y: event.clientY,
    original: cloneLayout(current.value), active: false }
}

function splitBounds({ index, sizes, minima }) {
  const total = sizes[index] + sizes[index + 1]
  const factor = Math.min(1, total / (minima[index] + minima[index + 1] || 1))
  const min = Math.min(total, minima[index] * factor)
  return { total, min, max: Math.max(min, total - minima[index + 1] * factor) }
}
function splitterAria(splitter) {
  const { total, min, max } = splitBounds(splitter)
  const position = splitter.sizes[splitter.index]
  const percent = (value) => total > 0 ? Math.round(1000 * value / total) / 10 : 0
  const dimensions = splitter.axis === 'row' ? ['Left width', 'right width'] : ['Top height', 'bottom height']
  return {
    'aria-valuemin': percent(min),
    'aria-valuemax': percent(max),
    'aria-valuenow': percent(Math.max(min, Math.min(max, position))),
    'aria-valuetext': `${dimensions[0]} ${Math.round(position)} pixels; ${dimensions[1]} ${Math.round(total - position)} pixels`,
  }
}
function resizeSplit(splitter, delta) {
  const { node, index } = splitter
  const sizes = [...splitter.sizes]
  const { total, min, max } = splitBounds(splitter)
  sizes[index] = Math.max(min, Math.min(max, sizes[index] + delta))
  sizes[index + 1] = total - sizes[index]
  // Start from displayed sizes so minimum-size projection cannot move other dividers.
  const available = sum(sizes) || 1
  node.children.forEach((child, i) => { child.weight = Math.max(1e-12, sizes[i] / available) })
}
function splitKey(event, splitter) {
  if (gesture.value) return
  const keys = splitter.axis === 'row' ? ['ArrowLeft', 'ArrowRight'] : ['ArrowUp', 'ArrowDown']
  if (!keys.includes(event.key)) return
  event.preventDefault()
  resizeSplit(splitter, event.key === keys[0] ? -10 : 10)
  publish(current.value)
}

async function move(event) {
  let edit = gesture.value
  if (!edit || event.pointerId !== edit.pointerId) return
  const dx = event.clientX - edit.x, dy = event.clientY - edit.y
  if (!edit.active && Math.hypot(dx, dy) < 6) return
  if (!edit.active) {
    edit = gesture.value = { ...edit, active: true }
    if (!edit.splitter) {
      // Header insertion needs the remaining panes' DOM as well as geometry.
      await nextTick()
      if (gesture.value !== edit) return
    }
  }
  if (edit.splitter) {
    resizeSplit(edit.splitter, edit.splitter.axis === 'row' ? dx : dy)
    return
  }
  updateDrop(event, edit)
}

function updateDrop(event, edit) {
  const rect = host.value.getBoundingClientRect()
  const x = event.clientX - rect.left, y = event.clientY - rect.top
  ghost.value = { left: x + 12, top: y + 12, width: Math.min(300, rect.width), height: Math.min(200, rect.height) }
  drop.value = null
  if (x < 0 || y < 0 || x > rect.width || y > rect.height) return
  if (current.value.root?.type === 'stack' && current.value.root.viewers.length === 1) return
  // A narrow band at the outer border wins over individual pane targets.
  // Choose the nearest edge at corners, so all four remain reachable.
  const edges = [['left', x], ['right', rect.width - x], ['top', y], ['bottom', rect.height - y]]
  edges.sort((a, b) => a[1] - b[1])
  if (edges[0][1] < Math.min(8, rect.width / 8, rect.height / 8)) {
    const side = edges[0][0]
    const preview = { left: 0, top: 0, width: rect.width, height: rect.height }
    if (side === 'left' || side === 'right') {
      preview.width /= 2
      if (side === 'right') preview.left = preview.width
    } else {
      preview.height /= 2
      if (side === 'bottom') preview.top = preview.height
    }
    drop.value = { root: true, side, preview }
    return
  }
  const pane = geometry.value.panes.find((box) => x >= box.left && x <= box.left + box.width && y >= box.top && y <= box.top + box.height)
  drop.value = null
  if (!pane || (pane.node.viewers.length === 1 && pane.node.viewers[0] === edit.id)) return
  let side = 'center', index, tab
  const preview = { left: pane.left, top: pane.top, width: pane.width, height: pane.height }
  const contentTop = pane.top + (props.hasHeaders ? HEADER : 0)
  const verticalEdge = (pane.top + pane.height - contentTop) / 4
  if (y < contentTop) {
    const container = tabElements.get(pane.key)
    const tabs = []
    let shift = 0
    // Measure without the rendered gap so insertion cannot oscillate. Read
    // the DOM rather than drop state: a same-turn release can precede rendering.
    for (const element of container.children) {
      const box = element.getBoundingClientRect()
      if (element.classList.contains('jdz-native-tab-placeholder')) shift += box.width
      else tabs.push({ left: box.left - shift, width: box.width })
    }
    index = tabs.findIndex((box) => event.clientX < box.left + box.width / 2)
    if (index < 0) index = tabs.length
    tab = { paneKey: pane.key, index }
    const insertionX = index < tabs.length ? tabs[index].left : tabs.at(-1).left + tabs.at(-1).width
    const viewport = container.getBoundingClientRect()
    preview.left = Math.max(viewport.left, Math.min(insertionX, viewport.right - TAB_DROP_WIDTH)) - rect.left
    preview.width = Math.min(TAB_DROP_WIDTH, viewport.width)
    // dockViewer accepts insertion indices from before source removal.
    const source = findStack(current.value.root, edit.id).node
    if (source.viewers.includes(pane.node.viewers[0]) && source.viewers.indexOf(edit.id) <= index) index++
    preview.height = HEADER
  } else if (x - pane.left < pane.width / 4) side = 'left'
  else if (pane.left + pane.width - x < pane.width / 4) side = 'right'
  else if (y - contentTop < verticalEdge) side = 'top'
  else if (pane.top + pane.height - y < verticalEdge) side = 'bottom'
  if (['left', 'right'].includes(side)) { preview.width /= 2; if (side === 'right') preview.left += preview.width }
  if (['top', 'bottom'].includes(side)) { preview.height /= 2; if (side === 'bottom') preview.top += preview.height }
  drop.value = { targetId: pane.node.viewers[0], side, index, tab, preview }
}

async function finish(event) {
  const edit = gesture.value
  if (!edit || edit.pointerId !== event.pointerId) return
  if (edit.active) {
    if (edit.splitter) {
      resizeSplit(edit.splitter, edit.splitter.axis === 'row' ? event.clientX - edit.x : event.clientY - edit.y)
      publish(current.value)
    } else {
      // Detaching the tab may still be rendering the remaining headers on release.
      await nextTick()
      if (gesture.value !== edit) return
      updateDrop(event, edit)
      if (drop.value) publish(drop.value.root
        ? dockViewerAtRoot(current.value, edit.id, drop.value.side)
        : dockViewer(current.value, edit.id, drop.value.targetId, drop.value.side, drop.value.index))
      else current.value = edit.original
    }
  } else if (!edit.splitter) select(edit.id)
  gesture.value = drop.value = null
  if (host.value.hasPointerCapture(event.pointerId)) host.value.releasePointerCapture(event.pointerId)
}
function cancel() {
  const edit = gesture.value
  gesture.value = drop.value = null
  if (edit) {
    current.value = edit.original
    if (host.value.hasPointerCapture(edit.pointerId)) host.value.releasePointerCapture(edit.pointerId)
  }
}
function lostCapture(event) {
  // Lab can lose capture just before pointerup reaches a sibling plot widget.
  if (event.pointerId === gesture.value?.pointerId && event.buttons) cancel()
}
function escape(event) { if (event.key === 'Escape') cancel() }

watch(() => props.layout, (layout) => {
  const next = normalizeLayout(layout)
  const acknowledged = pendingLayouts.indexOf(JSON.stringify(next))
  if (acknowledged !== -1) {
    pendingLayouts.splice(0, acknowledged + 1)
    return // A Python echo must not cancel or rewind a newer local edit.
  }
  pendingLayouts.length = 0
  cancel()
  current.value = next
}, { deep: true })
// A rejected edit can leave Python's layout unchanged. Its reset counter makes
// that rejection observable even when the canonical tree did not change.
watch(() => props.layoutReset, () => {
  pendingLayouts.length = 0
  cancel()
  current.value = normalizeLayout(props.layout)
})
function applyPlacement() {
  if (disposed || !host.value) return
  // Clip widgets immediately, even while plots are still redrawing. Registry
  // and layout messages may arrive separately; new host refs also trigger this.
  const placements = new Map()
  for (const pane of geometry.value.panes) {
    const header = props.hasHeaders ? HEADER : 0
    const width = Math.max(0, pane.width - 2), height = Math.max(0, pane.height - header - 2)
    placements.set(pane.node.activeViewerId, {
      ...boxStyle({ left: pane.left + 1, top: pane.top + header + 1, width, height }),
      display: width > 0 && height > 0 ? 'block' : 'none', zIndex: '1',
    })
  }
  let changed = false
  for (const [id, element] of viewerElements) {
    const style = placements.get(id) || { display: 'none' }
    const placement = { ...style, widget: style.display === 'none' ? null : metadata.value.get(id)?.widget }
    const previous = appliedPlacements.get(element)
    // Compare our own values; browsers round fractional pixel styles when read back.
    if (!previous || Object.entries(placement).some(([key, value]) => previous[key] !== value)) {
      Object.assign(element.style, style)
      appliedPlacements.set(element, placement)
      changed = true
    }
  }
  updateOverflow()
  // Header and unrelated metadata updates do not require plot resize work.
  if (changed) {
    cancelAnimationFrame(frame)
    frame = requestAnimationFrame(() => emit('layout-applied'))
  }
}
function schedulePlacement() {
  if (placementQueued || disposed) return
  placementQueued = true
  nextTick(() => {
    placementQueued = false
    applyPlacement()
  })
}
watch(geometry, schedulePlacement, { deep: true, flush: 'post' })
// Only header changes and widget replacements need work beyond geometry updates.
watch(() => props.hasHeaders, schedulePlacement, { flush: 'post' })
watch(() => props.viewers.map(({ id, widget, reference, name, closable }) =>
  [id, widget, reference, name, closable]), schedulePlacement, { flush: 'post' })
onMounted(() => {
  observer = new ResizeObserver(() => {
    const width = host.value.clientWidth, height = host.value.clientHeight
    if (width !== size.value.width || height !== size.value.height) size.value = { width, height }
    schedulePlacement()
  })
  observer.observe(host.value)
  for (const element of tabElements.values()) observer.observe(element)
  schedulePlacement()
  window.addEventListener('keydown', escape)
  window.addEventListener('blur', cancel)
  window.addEventListener('pointerup', finish, true)
})
onBeforeUnmount(() => {
  disposed = true
  cancel()
  cancelAnimationFrame(frame)
  observer.disconnect()
  window.removeEventListener('keydown', escape)
  window.removeEventListener('blur', cancel)
  window.removeEventListener('pointerup', finish, true)
  for (const element of viewerElements.values()) element.style.display = 'none'
})
</script>

<style>
.jdz-viewer-layout { position: relative; width: 100%; height: 100%; min-width: 0; min-height: 0; overflow: hidden; }
.jdz-native-layout {
  --jdz-dock-surface: #fff; --jdz-dock-header: #eee; --jdz-dock-ink: #333;
  --jdz-dock-border: #bbb; --jdz-dock-hover: #ddd; --jdz-dock-accent: #4477aa;
  color: var(--jdz-dock-ink); font: 13px sans-serif;
}
.theme--dark .jdz-native-layout, .v-theme--dark .jdz-native-layout {
  --jdz-dock-surface: #212121; --jdz-dock-header: #303030; --jdz-dock-ink: #eee;
  --jdz-dock-border: #666; --jdz-dock-hover: #454545; --jdz-dock-accent: #90caf9;
}
.jdz-viewer-layout__viewers { position: absolute; inset: 0; pointer-events: none; }
.jdz-viewer-layout__viewer { position: absolute; overflow: hidden; display: none; pointer-events: auto; box-sizing: border-box; min-width: 0; min-height: 0; }
.jdz-viewer-layout__viewer > * { width: 100%; height: 100%; }
.jdz-native-pane { position: absolute; box-sizing: border-box; border: 1px solid var(--jdz-dock-border); background: var(--jdz-dock-surface); }
.jdz-native-header { height: 27px; display: flex; background: var(--jdz-dock-header); user-select: none; }
.jdz-native-tabs { display: flex; flex: 1; min-width: 0; overflow-x: auto; scrollbar-width: thin; }
.jdz-native-tab { display: flex; flex-shrink: 0; align-items: center; padding: 2px 3px; white-space: nowrap; border-right: 1px solid var(--jdz-dock-border); }
.jdz-native-tab-placeholder { flex: 0 0 100px; }
.jdz-native-tab.active { background: var(--jdz-dock-surface); box-shadow: inset 0 2px var(--jdz-dock-accent); }
.jdz-native-layout button:focus-visible, .jdz-native-overflow:focus-visible, .jdz-native-splitter:focus-visible { outline: 2px solid var(--jdz-dock-accent); outline-offset: -2px; }
.jdz-native-layout button { border: 0; padding: 0 4px; background: transparent; color: inherit; font: inherit; cursor: pointer; }
.jdz-native-layout .jdz-native-tab-label { height: 100%; cursor: grab; touch-action: none; }
.jdz-native-layout button:hover { background: var(--jdz-dock-hover); }
.jdz-native-overflow { flex: 0 0 28px; width: 28px; appearance: auto; color: inherit; background: var(--jdz-dock-header); cursor: pointer; }
.jdz-native-overflow option { color: var(--jdz-dock-ink); background: var(--jdz-dock-surface); }
.jdz-native-layout .jdz-native-maximize { flex: 0 0 25px; font-size: 18px; }
.jdz-native-splitter { position: absolute; z-index: 2; touch-action: none; background: var(--jdz-dock-hover); }
.jdz-native-splitter.jdz-native-row { cursor: col-resize; }
.jdz-native-splitter.jdz-native-column { cursor: row-resize; }
.jdz-native-splitter:hover { background: var(--jdz-dock-accent); }
.jdz-native-drag-shield { position: absolute; inset: 0; z-index: 3; cursor: grabbing; }
.jdz-native-ghost { position: absolute; z-index: 4; pointer-events: none; border: 1px solid var(--jdz-dock-accent); box-sizing: border-box; background: color-mix(in srgb, var(--jdz-dock-surface) 75%, transparent); box-shadow: 2px 2px 4px #0003; }
.jdz-native-ghost-title { height: 27px; box-sizing: border-box; padding: 5px 8px; overflow: hidden; white-space: nowrap; text-overflow: ellipsis; background: var(--jdz-dock-header); border-bottom: 1px solid var(--jdz-dock-border); }
.jdz-native-drop { position: absolute; z-index: 5; pointer-events: none; background: color-mix(in srgb, var(--jdz-dock-accent) 18%, transparent); border: 1px dashed var(--jdz-dock-accent); box-sizing: border-box; }
.jdz-native-drop-tab { border-style: solid; }
</style>
