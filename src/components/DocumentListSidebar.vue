<script>
    import { mapStores, mapState } from 'pinia'
    import { useHeynoteStore } from "@/src/stores/heynote-store"
    import { useSettingsStore } from "@/src/stores/settings-store"
    import { getBlocks, getActiveNoteBlock } from "@/src/editor/block/block.js"

    const MIN_WIDTH = 140
    const MAX_WIDTH = 400
    const DEFAULT_WIDTH = 200

    function clampWidth(w) {
        return Math.min(MAX_WIDTH, Math.max(MIN_WIDTH, w))
    }

    function blockPreview(state, block, maxLen = 24) {
        const line = state.doc.sliceString(block.content.from, block.content.to).split('\n')[0] || ''
        const trimmed = line.trim()
        return trimmed.length > maxLen ? trimmed.slice(0, maxLen) + '…' : trimmed
    }

    export default {
        name: 'DocumentListSidebar',
        data() {
            return {
                dragging: false,
                resizeStartX: 0,
                resizeStartWidth: 0,
                dragWidth: 0,
                expandedPath: null,
            }
        },
        computed: {
            ...mapStores(useHeynoteStore, useSettingsStore),
            ...mapState(useHeynoteStore, ['openTabs', 'currentBufferPath', 'blockListVersion']),
            storedWidth() {
                const w = this.settingsStore.settings.documentListSidebarWidth
                return typeof w === 'number' && w > 0 ? w : DEFAULT_WIDTH
            },
            sidebarWidth() {
                if (this.dragging) {
                    return clampWidth(this.dragWidth)
                }
                return clampWidth(this.storedWidth)
            },
            items() {
                return this.openTabs.map((path) => ({
                    path,
                    title: this.heynoteStore.getBufferTitle(path),
                    active: path === this.currentBufferPath,
                    expanded: this.expandedPath === path,
                }))
            },
            expandedBlocks() {
                this.blockListVersion // reactive dependency for doc changes
                if (this.expandedPath !== this.currentBufferPath) return []
                const editor = this.heynoteStore.currentEditor
                if (!editor?.view?.state) return []
                const state = editor.view.state
                const rawBlocks = getBlocks(state)
                const active = getActiveNoteBlock(state)
                return rawBlocks.map((block, index) => {
                    const isEmpty = block.content.from === block.content.to
                    const preview = blockPreview(state, block)
                    const label = isEmpty ? '(empty)' : (preview || block.language?.name || 'text')
                    return { index, block, label, active: active === block }
                })
            },
        },
        methods: {
            onSelect(path) {
                this.heynoteStore.openBuffer(path)
                this.expandedPath = path
                this.heynoteStore.focusEditor()
            },
            onExpandClick(path, event) {
                event.stopPropagation()
                this.expandedPath = this.expandedPath === path ? null : path
                if (this.expandedPath && path !== this.currentBufferPath) this.onSelect(path)
            },
            goToBlock(block, event) {
                event.stopPropagation()
                const editor = this.heynoteStore.currentEditor
                if (!editor?.view?.state) return
                const state = editor.view.state
                const { from, to } = block.content
                const pos = to > from ? state.doc.lineAt(to - 1).to : from
                editor.setCursorPosition(pos)
                this.heynoteStore.focusEditor()
            },
            onClose(path, event) {
                event.stopPropagation()
                if (this.expandedPath === path) this.expandedPath = null
                this.heynoteStore.closeTab(path)
            },
            onContextMenu(path, event) {
                event.preventDefault()
                window.heynote?.mainProcess?.invoke?.('showTabContextMenu', path)
            },
            startResize(event) {
                if (event.button !== 0) return
                this.dragging = true
                this.resizeStartX = event.clientX
                this.resizeStartWidth = this.sidebarWidth
                this.dragWidth = this.resizeStartWidth
                document.addEventListener('mousemove', this.onResize)
                document.addEventListener('mouseup', this.endResize)
            },
            onResize(event) {
                if (!this.dragging) return
                const delta = event.clientX - this.resizeStartX
                this.dragWidth = clampWidth(this.resizeStartWidth + delta)
            },
            endResize() {
                if (!this.dragging) return
                this.dragging = false
                document.removeEventListener('mousemove', this.onResize)
                document.removeEventListener('mouseup', this.endResize)
                this.settingsStore.updateSettings({ documentListSidebarWidth: clampWidth(this.dragWidth) })
            },
        },
    }
</script>

<template>
    <aside class="document-list-sidebar" :style="{ width: sidebarWidth + 'px' }">
        <div class="sidebar-header">Documents</div>
        <ul class="document-tree">
            <template v-for="item in items" :key="item.path">
                <li class="tree-node tree-node-document">
                    <div
                        :class="['node-row', 'document-row', { active: item.active }]"
                        :title="item.title"
                        @click="onSelect(item.path)"
                        @contextmenu="onContextMenu(item.path, $event)"
                    >
                        <button
                            type="button"
                            class="expand"
                            :class="{ expanded: item.expanded }"
                            :aria-label="item.expanded ? 'Collapse' : 'Expand'"
                            tabindex="-1"
                            @click="onExpandClick(item.path, $event)"
                        />
                        <span class="document-title">{{ item.title }}</span>
                        <button
                            type="button"
                            class="close"
                            tabindex="-1"
                            aria-label="Close"
                            @click="onClose(item.path, $event)"
                        />
                    </div>
                    <ul
                        v-if="item.expanded && item.path === currentBufferPath"
                        class="tree-children"
                    >
                        <li
                            v-for="(entry, idx) in expandedBlocks"
                            :key="entry.index"
                            class="tree-node tree-node-block"
                        >
                            <div
                                :class="['node-row', 'block-row', { active: entry.active }]"
                                :title="entry.label"
                                @click="goToBlock(entry.block, $event)"
                            >
                                <span class="tree-prefix">{{ idx === expandedBlocks.length - 1 ? '└' : '├' }}── </span>
                                <span class="block-label">{{ entry.label }}</span>
                            </div>
                        </li>
                    </ul>
                </li>
            </template>
        </ul>
        <div
            class="resize-handle"
            aria-label="Resize sidebar"
            @mousedown.prevent="startResize"
        />
    </aside>
</template>

<style lang="sass" scoped>
.document-list-sidebar
    flex-shrink: 0
    position: relative
    min-width: 140px
    max-width: 400px
    display: flex
    flex-direction: column
    background: var(--panel-background)
    border-right: 1px solid var(--tab-bar-border-bottom-color)
    overflow: hidden

.sidebar-header
    flex-shrink: 0
    padding: 8px 12px
    font-size: 12px
    font-weight: 600
    color: rgba(0, 0, 0, 0.7)
    border-bottom: 1px solid var(--tab-bar-border-bottom-color)
    +dark-mode
        color: rgba(255, 255, 255, 0.7)

.document-tree
    flex: 1
    list-style: none
    margin: 0
    padding: 6px 8px
    overflow-y: auto
    min-width: 0

.tree-node
    list-style: none
    margin: 0
    padding: 0
    min-width: 0

.node-row
    display: flex
    align-items: center
    min-width: 0
    cursor: pointer
    user-select: none
    overflow: hidden
    border-top: 1px solid transparent
    border-bottom: 1px solid transparent

.document-row
    min-height: 28px
    padding: 5px 8px
    margin-bottom: 2px
    line-height: 1.4
    color: rgba(0, 0, 0, 0.6)
    +dark-mode
        color: rgba(255, 255, 255, 0.5)
    &:hover
        background: #e0e0e0
        color: rgba(0, 0, 0, 0.9)
        .close
            display: block
        +dark-mode
            background: #3a3a3a
            color: rgba(255, 255, 255, 0.9)
    &.active
        background: var(--tab-active-bg)
        color: rgba(0, 0, 0, 0.9)
        border-color: var(--tab-active-bg)
        .close
            display: block
        +dark-mode
            color: rgba(255, 255, 255, 0.85)
            background: var(--tab-active-bg)
            border-color: var(--tab-active-bg)

.expand
    flex-shrink: 0
    width: 20px
    height: 20px
    margin-right: 2px
    padding: 0
    border: none
    background: none
    background-image: url("@/assets/icons/caret-right.svg")
    background-repeat: no-repeat
    background-size: 12px
    background-position: center
    cursor: pointer
    opacity: 0.8
    +dark-mode
        background-image: url("@/assets/icons/caret-right-white.svg")
    &:hover
        opacity: 1
    &.expanded
        background-image: url("@/assets/icons/caret-down.svg")
        +dark-mode
            background-image: url("@/assets/icons/caret-down-white.svg")

.document-title
    flex: 1
    min-width: 0
    font-size: 12px
    line-height: 1.4
    white-space: nowrap
    overflow: hidden
    text-overflow: ellipsis
    padding: 0 4px

.close
    display: none
    flex-shrink: 0
    width: 20px
    height: 20px
    margin-left: 2px
    padding: 0
    border: none
    background: none
    background-image: url("@/assets/icons/close.svg")
    background-repeat: no-repeat
    background-size: 14px
    background-position: center
    border-radius: 3px
    cursor: pointer
    opacity: 0.7
    &:hover
        background-color: rgba(0, 0, 0, 0.15)
        opacity: 1
        +dark-mode
            background-color: rgba(255, 255, 255, 0.15)

.tree-children
    list-style: none
    margin: 0 0 6px 0
    padding: 0 0 0 8px
    min-width: 0

.tree-node-block
    margin: 0
    padding: 0

.block-row
    padding: 2px 4px 2px 2px
    margin-left: 12px
    font-size: 11px
    line-height: 1.45
    min-width: 0
    white-space: nowrap
    overflow: hidden
    text-overflow: ellipsis
    border-radius: 3px
    border: 1px solid transparent
    color: rgba(0, 0, 0, 0.55)
    +dark-mode
        color: rgba(255, 255, 255, 0.5)
    &:hover
        background: rgba(0, 0, 0, 0.06)
        color: rgba(0, 0, 0, 0.75)
        +dark-mode
            background: rgba(255, 255, 255, 0.08)
            color: rgba(255, 255, 255, 0.8)
    &.active
        background: rgba(0, 0, 0, 0.08)
        color: rgba(0, 0, 0, 0.9)
        font-weight: 500
        border-color: rgba(0, 0, 0, 0.08)
        +dark-mode
            background: rgba(255, 255, 255, 0.12)
            color: rgba(255, 255, 255, 0.9)
            border-color: rgba(255, 255, 255, 0.12)

.tree-prefix
    flex-shrink: 0
    color: rgba(0, 0, 0, 0.35)
    +dark-mode
        color: rgba(255, 255, 255, 0.4)

.block-label
    flex: 1
    min-width: 0
    overflow: hidden
    text-overflow: ellipsis
    padding-right: 2px

.resize-handle
    position: absolute
    top: 0
    right: 0
    width: 6px
    height: 100%
    cursor: col-resize
    z-index: 1
    &:hover
        background: rgba(0, 0, 0, 0.08)
        +dark-mode
            background: rgba(255, 255, 255, 0.08)
    &:active
        background: rgba(0, 0, 0, 0.12)
        +dark-mode
            background: rgba(255, 255, 255, 0.12)
</style>
