<!-- @format -->

<template>
  <div class="index-page" :class="{ 'index-page--mobile': isMobile }" v-loading="isLoading">
    <HeaderNav
      :active-mode="editorMode"
      :active-preview-mode="svPreviewMode"
      :toolbar-hidden="isMobile ? toolbarHidden : false"
      @toggle-sidebar="onToggleSidebar"
      @switch-mode="onSwitchMode"
      @preview-action="onPreviewAction"
      @clear-content="onClearContent"
      @toggle-toolbar="onToggleToolbar"
    />
    <div class="index-page__body">
      <Sidebar
        v-if="!isMobile || !sidebarCollapsed"
        :collapsed="isMobile ? false : sidebarCollapsed"
        :is-mobile="isMobile"
        :active-doc-id="activeDocId"
        @select-doc="onSelectDoc"
        @doc-deleted="onDocDeleted"
        @toggle-sidebar="onToggleSidebar"
      />
      <div
        v-if="isMobile && !sidebarCollapsed"
        class="index-page__sidebar-overlay"
        @click="onToggleSidebar"
      />
      <div
        class="index-page__editor"
        :style="{ marginLeft: isMobile ? 0 : sidebarCollapsed ? '48px' : '260px' }"
      >
        <div :key="editorRenderKey" id="vditor" class="vditor" />
      </div>
    </div>
  </div>
</template>

<script>
import Vditor from 'vditor'
import 'vditor/src/assets/less/index.less'
import HeaderNav from './partials/HeaderNav'
import Sidebar from '@components/Sidebar'
import { MessageBox } from 'element-ui'
import defaultText from '@config/default'
import {
  migrateFromLegacy,
  getDocuments,
  getActiveDocId,
  setActiveDocId,
  getDocContent,
  saveDocContent,
} from '@helper/storage'
import { trackEvent } from '@helper/analytics'

const SAVE_DEBOUNCE_MS = 1000
const MOBILE_BREAKPOINT = 1200
const detectMobileViewport = () => {
  const hasCoarsePointer =
    typeof window.matchMedia === 'function' && window.matchMedia('(pointer: coarse)').matches
  return window.innerWidth <= MOBILE_BREAKPOINT || hasCoarsePointer
}

export default {
  name: 'index-page',

  data() {
    return {
      isLoading: true,
      isMobile: detectMobileViewport(),
      vditor: null,
      activeDocId: null,
      sidebarCollapsed: detectMobileViewport(),
      editorMode: detectMobileViewport() ? 'wysiwyg' : 'sv',
      svPreviewMode: 'both',
      toolbarHidden: detectMobileViewport(),
      editorRenderKey: 0,
      suppressAutoFocus: false,
      saveTimer: null,
    }
  },

  created() {
    migrateFromLegacy(defaultText)
    if (getDocuments().length === 0) {
      const { createDocument } = require('@helper/storage')
      createDocument('未命名文档')
    }
    this.activeDocId = getActiveDocId() || (getDocuments()[0] && getDocuments()[0].id)
    if (this.activeDocId) {
      setActiveDocId(this.activeDocId)
    }
    console.log = () => {}
  },

  components: {
    HeaderNav,
    Sidebar,
  },

  mounted() {
    this.initVditor()
    this.$nextTick(() => {
      this.isLoading = false
    })
    this.$root.$on('reload-content', this.reloadContent)
    window.addEventListener('resize', this.onResize)
  },

  beforeDestroy() {
    this.saveCurrentDoc()
    this.$root.$off('reload-content', this.reloadContent)
    window.removeEventListener('resize', this.onResize)
    if (this.saveTimer) clearTimeout(this.saveTimer)
  },

  methods: {
    onResize() {
      const nextIsMobile = detectMobileViewport()
      if (nextIsMobile !== this.isMobile) {
        this.isMobile = nextIsMobile
        this.sidebarCollapsed = nextIsMobile
        this.toolbarHidden = nextIsMobile
        this.updateToolbarConfig()
        return
      }
      this.isMobile = nextIsMobile
      this.updateToolbarConfig()
    },
    getToolbarConfig() {
      return {
        hide: this.isMobile && this.toolbarHidden,
        pin: !this.isMobile,
      }
    },
    updateToolbarConfig() {
      if (!this.vditor || typeof this.vditor.updateToolbarConfig !== 'function') return
      this.vditor.updateToolbarConfig(this.getToolbarConfig())
    },
    onToggleSidebar() {
      this.sidebarCollapsed = !this.sidebarCollapsed
      trackEvent('sidebar_toggle', 'sidebar', this.sidebarCollapsed ? 'collapse' : 'expand')
    },
    focusEditorIfNeeded() {
      if (!this.vditor || typeof this.vditor.focus !== 'function') return
      if (this.isMobile) return
      this.vditor.focus()
    },
    blurActiveElementIfMobile() {
      if (!this.isMobile) return
      const activeElement = document.activeElement
      if (activeElement && typeof activeElement.blur === 'function') {
        activeElement.blur()
      }
    },
    initVditor(initialContent = null) {
      const that = this
      const options = {
        width: '100%',
        height: '0',
        tab: '\t',
        counter: '999999',
        typewriterMode: !this.isMobile,
        mode: this.editorMode,
        toolbarConfig: this.getToolbarConfig(),
        cache: { enable: false },
        preview: {
          delay: 100,
          show: this.editorMode === 'sv',
          mode: this.svPreviewMode,
        },
        outline: this.editorMode === 'sv' && !this.isMobile,
        upload: {
          max: 5 * 1024 * 1024,
          handler(file) {
            let formData = new FormData()
            for (let i in file) {
              formData.append('smfile', file[i])
            }
            let request = new XMLHttpRequest()
            request.open('POST', 'https://sm.ms/api/upload')
            request.onload = that.onloadCallback
            request.send(formData)
          },
        },
        input: (value) => {
          that.debouncedSave(value)
        },
        after: () => {
          const content = initialContent == null ? getDocContent(this.activeDocId) || defaultText : initialContent
          this.vditor.setValue(content)
          if (this.editorMode === 'sv' && this.vditor && typeof this.vditor.setPreviewMode === 'function') {
            this.vditor.setPreviewMode(this.svPreviewMode)
          }
          if (!this.suppressAutoFocus) {
            this.focusEditorIfNeeded()
          }
          this.suppressAutoFocus = false
        },
      }
      this.vditor = new Vditor('vditor', options)
    },
    onToggleToolbar() {
      if (!this.isMobile) return
      this.toolbarHidden = !this.toolbarHidden
      this.updateToolbarConfig()
      trackEvent('editor_toolbar_toggle', 'editor', this.toolbarHidden ? 'hide' : 'show')
    },
    onSwitchMode(mode) {
      let nextMode = this.editorMode
      let nextSVPreviewMode = this.svPreviewMode
      if (mode === 'wysiwyg') {
        nextMode = 'wysiwyg'
      } else if (mode === 'sv-both') {
        nextMode = 'sv'
        nextSVPreviewMode = 'both'
      } else if (mode === 'sv-editor') {
        nextMode = 'sv'
        nextSVPreviewMode = 'editor'
      } else if (['ir', 'sv'].includes(mode)) {
        nextMode = mode
      } else {
        return
      }

      if (nextMode === this.editorMode && nextSVPreviewMode === this.svPreviewMode) return
      if (nextMode === 'sv' && this.editorMode === 'sv' && nextSVPreviewMode !== this.svPreviewMode) {
        this.svPreviewMode = nextSVPreviewMode
        if (this.vditor && typeof this.vditor.setPreviewMode === 'function') {
          this.vditor.setPreviewMode(this.svPreviewMode)
        }
        trackEvent('editor_sv_preview_switch', 'editor', this.svPreviewMode)
        return
      }

      const currentValue =
        this.vditor && typeof this.vditor.getValue === 'function'
          ? this.vditor.getValue()
          : getDocContent(this.activeDocId) || ''
      this.saveCurrentDoc()
      this.blurActiveElementIfMobile()
      if (this.vditor && typeof this.vditor.destroy === 'function') {
        this.vditor.destroy()
        this.vditor = null
      }
      this.suppressAutoFocus = true
      this.editorMode = nextMode
      this.svPreviewMode = nextSVPreviewMode
      this.editorRenderKey += 1
      this.$nextTick(() => {
        this.initVditor(currentValue)
      })
      trackEvent(
        'editor_mode_switch',
        'editor',
        this.editorMode === 'sv' ? `${this.editorMode}:${this.svPreviewMode}` : this.editorMode
      )
    },
    onPreviewAction(type) {
      if (type === 'copy-plain-text') {
        this.copyPlainTextForChat()
        return
      }
      if (this.editorMode !== 'sv') return
      const target = document.querySelector(`.vditor-preview__action button[data-type="${type}"]`)
      if (!target || typeof target.click !== 'function') return
      target.click()
      trackEvent('editor_preview_action', 'editor', type)
    },
    copyByExecCommand(text) {
      const textarea = document.createElement('textarea')
      textarea.value = text
      textarea.setAttribute('readonly', 'readonly')
      textarea.style.position = 'fixed'
      textarea.style.left = '-9999px'
      textarea.style.top = '-9999px'
      document.body.appendChild(textarea)
      textarea.select()
      let copied = false
      try {
        copied = document.execCommand('copy')
      } catch (error) {
        copied = false
      }
      document.body.removeChild(textarea)
      return copied
    },
    copyPlainTextForChat() {
      if (!this.vditor) return
      let plainText = ''
      if (typeof this.vditor.getHTML === 'function') {
        const html = this.vditor.getHTML() || ''
        const textHolder = document.createElement('div')
        textHolder.innerHTML = html
        plainText = (textHolder.innerText || textHolder.textContent || '')
          .replace(/\r\n/g, '\n')
          .replace(/\n{3,}/g, '\n\n')
          .trim()
      }
      if (!plainText && typeof this.vditor.getValue === 'function') {
        plainText = String(this.vditor.getValue() || '').trim()
      }
      if (!plainText) {
        this.$message({
          type: 'warning',
          message: '没有可复制的内容',
        })
        return
      }

      const onSuccess = () => {
        this.$message({
          type: 'success',
          message: '已复制纯文本，可直接粘贴到微信聊天',
        })
        trackEvent('editor_copy_plain_text', 'editor', 'success')
      }
      const onFail = () => {
        this.$message({
          type: 'error',
          message: '复制失败，请手动复制',
        })
        trackEvent('editor_copy_plain_text', 'editor', 'fail')
      }

      if (navigator.clipboard && window.isSecureContext) {
        navigator.clipboard
          .writeText(plainText)
          .then(onSuccess)
          .catch(() => {
            this.copyByExecCommand(plainText) ? onSuccess() : onFail()
          })
      } else {
        this.copyByExecCommand(plainText) ? onSuccess() : onFail()
      }
    },
    onClearContent() {
      if (!this.vditor) return
      MessageBox.confirm('确定要清空当前文档内容吗？', '清空内容', {
        confirmButtonText: '清空',
        cancelButtonText: '取消',
        type: 'warning',
      })
        .then(() => {
          this.vditor.setValue('')
          if (this.activeDocId) {
            saveDocContent(this.activeDocId, '')
          }
          this.focusEditorIfNeeded()
          trackEvent('editor_clear_confirm', 'editor', this.activeDocId || '')
        })
        .catch(() => {
          trackEvent('editor_clear_cancel', 'editor', this.activeDocId || '')
        })
    },
    debouncedSave(value) {
      if (this.saveTimer) clearTimeout(this.saveTimer)
      this.saveTimer = setTimeout(() => {
        if (this.activeDocId && this.vditor && typeof this.vditor.getValue === 'function') {
          const content = this.vditor.getValue()
          saveDocContent(this.activeDocId, content)
        }
        this.saveTimer = null
      }, SAVE_DEBOUNCE_MS)
    },
    saveCurrentDoc() {
      if (this.saveTimer) {
        clearTimeout(this.saveTimer)
        this.saveTimer = null
      }
      if (this.activeDocId && this.vditor && typeof this.vditor.getValue === 'function') {
        saveDocContent(this.activeDocId, this.vditor.getValue())
      }
    },
    onSelectDoc(id) {
      this.saveCurrentDoc()
      setActiveDocId(id)
      this.activeDocId = id
      const content = getDocContent(id) || ''
      this.vditor.setValue(content)
      this.focusEditorIfNeeded()
      if (this.isMobile) this.sidebarCollapsed = true
    },
    onDocDeleted() {
      this.activeDocId = getActiveDocId()
      if (this.activeDocId && this.vditor) {
        this.vditor.setValue(getDocContent(this.activeDocId) || '')
        this.focusEditorIfNeeded()
      } else if (this.vditor) {
        this.vditor.setValue('')
      }
    },
    onloadCallback(oEvent) {
      const currentTarget = oEvent.currentTarget
      if (currentTarget.status !== 200) {
        trackEvent('editor_image_upload_error', 'editor', currentTarget.statusText)
        return this.$message({
          type: 'error',
          message: currentTarget.status + ' ' + currentTarget.statusText,
        })
      }
      let resp = JSON.parse(currentTarget.response)
      let imgMdStr = ''
      if (resp.code === 'invalid_source') {
        trackEvent('editor_image_upload_invalid', 'editor', resp.message)
        return this.$message({
          type: 'error',
          message: resp.message,
        })
      }
      if (resp.code === 'image_repeated') {
        imgMdStr = `![](${resp.images})`
      } else if (resp.code === 'success' || resp.success) {
        imgMdStr = `![${resp.data.filename}](${resp.data.url})`
      }
      this.vditor.insertValue(imgMdStr)
      trackEvent('editor_image_upload_success', 'editor', resp.data ? resp.data.filename : '')
    },
    reloadContent() {
      this.activeDocId = getActiveDocId()
      if (this.vditor && this.vditor.getValue) {
        const content = getDocContent(this.activeDocId) || ''
        this.vditor.setValue(content)
        this.focusEditorIfNeeded()
      }
    },
  },
}
</script>

<style lang="less">
@import './../assets/styles/style.less';

.index-page {
  width: 100%;
  min-height: 100vh;
  height: 100%;
  background-color: @white;
  display: flex;
  flex-direction: column;

  .index-page__body {
    flex: 1;
    display: flex;
    margin-top: @header-height;
    margin-left: auto;
    margin-right: auto;
    min-height: calc(100vh - @header-height);
    overflow: hidden;
    position: relative;
  }

  .index-page__sidebar-overlay {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    z-index: 99;
    background: rgba(0, 0, 0, 0.3);
    transition: opacity 0.2s ease;
  }

  .index-page__editor {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    padding: 2rem;
    max-width: @max-body-width;
    margin: 0 auto;
    width: 100%;
    transition: all 0.3s ease;
  }

  .vditor {
    flex: 1;
    height: 100%;
    min-height: calc(100vh - @header-height - 48px);
    text-align: left;
    overflow: hidden;

    .vditor-toolbar {
      border-bottom: none;
      background-color: #fcfcfc;
    }

    .vditor-content {
      height: auto;
      min-height: auto;
      border-top: none;
    }

    .vditor-preview__action {
      display: none;
    }
  }

  .vditor-reset {
    font-size: 14px;
  }

  .vditor-textarea {
    font-size: 14px;
    height: 100% !important;
  }
}

.index-page.index-page--mobile {
  width: 100%;
  max-width: 100vw;
  overflow-x: hidden;

  .index-page__body {
    width: 100%;
    max-width: 100vw;
    overflow-x: hidden;
  }

  .index-page__editor {
    padding: 0;
    max-width: 100%;
    width: 100%;
    overflow-x: hidden;
  }

  .vditor {
    min-height: calc(100vh - @header-height);
    border: none;
    width: 100%;
    max-width: 100vw;
    overflow-x: hidden;

    .vditor-toolbar {
      overflow-x: auto;
      overflow-y: hidden;
      white-space: nowrap;
      -webkit-overflow-scrolling: touch;
      padding: 0 6px;

      &.vditor-toolbar--hide {
        height: 0;
        padding-top: 0;
        padding-bottom: 0;
        border-bottom: 0;
        overflow: hidden;
      }

      &.vditor-toolbar--hide:hover {
        height: 0;
        overflow: hidden;
      }
    }

    .vditor-content {
      min-height: calc(100vh - @header-height - 52px);
      width: 100%;
      max-width: 100vw;
      overflow-x: hidden;
    }

    .vditor-toolbar.vditor-toolbar--hide + .vditor-content {
      min-height: calc(100vh - @header-height);
    }

  }

  .vditor-reset,
  .vditor-textarea {
    font-size: 16px;
  }
}
</style>
