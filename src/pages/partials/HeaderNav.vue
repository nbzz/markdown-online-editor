<template>
  <header class="header-wrapper">
    <h1 class="header-area">
      <a href="/" class="header-link" target="_self">
        <img
          class="mark-markdown"
          src="@assets/images/markdown.png"
          alt="Arya 在线 Markdown 编辑器 Logo"
        />
        <strong v-if="!isMobile" class="header-text">{{ titleText }}</strong>
      </a>
      <nav class="button-group">
        <template v-if="isMobile">
          <span class="hint--bottom header-link" @click="$emit('toggle-sidebar')" aria-label="文档列表">
            <icon class="header-icon" name="sidebar" />
          </span>
          <div class="mode-switch" aria-label="编辑模式切换">
            <button
              type="button"
              class="mode-switch__btn"
              :class="{ 'mode-switch__btn--active': activeMode === 'wysiwyg' }"
              @click="onModeSwitch('wysiwyg')"
            >
              可视
            </button>
            <button
              type="button"
              class="mode-switch__btn"
              :class="{ 'mode-switch__btn--active': activeMode === 'sv' && activePreviewMode === 'editor' }"
              @click="onModeSwitch('sv-editor')"
            >
              MD
            </button>
            <button
              type="button"
              class="mode-switch__btn"
              :class="{ 'mode-switch__btn--active': activeMode === 'sv' && activePreviewMode === 'both' }"
              @click="onModeSwitch('sv-both')"
            >
              分屏
            </button>
            <button type="button" class="mode-switch__btn mode-switch__btn--danger" @click="onClearContent">
              清空
            </button>
          </div>
          <el-dropdown trigger="click" @command="handleMobileCommand">
            <span class="hint--bottom el-dropdown-link" aria-label="更多功能">
              <icon class="header-icon" name="setting" />
            </span>
            <el-dropdown-menu slot="dropdown">
              <el-dropdown-item command="copy-plain-text">
                复制纯文本
              </el-dropdown-item>
              <el-dropdown-item command="toggle-toolbar">
                {{ toolbarHidden ? '展开编辑工具' : '折叠编辑工具' }}
              </el-dropdown-item>
              <el-dropdown-item command="import">导入文件</el-dropdown-item>
              <el-dropdown-item command="/export/ppt">预览 PPT</el-dropdown-item>
              <el-dropdown-item command="/export/png">导出 PNG</el-dropdown-item>
              <el-dropdown-item command="/export/pdf">导出 PDF</el-dropdown-item>
              <el-dropdown-item command="/about-arya">关于 Arya</el-dropdown-item>
              <el-dropdown-item command="fullscreen">全屏</el-dropdown-item>
            </el-dropdown-menu>
          </el-dropdown>
        </template>
        <template v-else>
          <a
            href="https://wechat.jeffjade.com/"
            class="header-link"
            target="_blank"
            rel="noopener"
          >
            <span class="hint--bottom" aria-label="公众号 Markdown 排版">
              <icon class="header-icon" name="wechat" />
            </span>
          </a>
          <a href="https://www.niceshare.site/" class="header-link" target="_blank" rel="noopener">
            <span class="hint--bottom" aria-label="逍遥自在轩">
              <icon class="header-icon" name="homepage" />
            </span>
          </a>
          <a href="https://www.lovejade.cn/" class="header-link" target="_blank" rel="noopener">
            <span class="hint--bottom" aria-label="清风明月轩">
              <icon class="header-icon" name="home" />
            </span>
          </a>
          <a
            href="https://x.com/MarshalXuan"
            class="header-link"
            target="_blank"
            rel="noopener"
          >
            <span class="hint--bottom" aria-label="X - 轩帅">
              <icon class="header-icon" name="x" />
            </span>
          </a>
          <a
            href="https://github.com/nicejade"
            class="header-link"
            target="_blank"
            rel="noopener"
          >
            <span class="hint--bottom" aria-label="作者 Github">
              <icon class="header-icon" name="github" />
            </span>
          </a>
          <router-link to="/about-arya" class="header-link">
            <span class="hint--bottom" aria-label="关于 Arya">
              <icon class="header-icon" name="document" />
            </span>
          </router-link>
          <span class="hint--bottom" @click="onImportClick" aria-label="导入文件">
            <icon class="header-icon" name="upload" />
          </span>
          <el-dropdown trigger="click" @command="handleCommand">
            <span class="hint--bottom el-dropdown-link" aria-label="设置">
              <icon class="header-icon" name="setting" />
            </span>
            <el-dropdown-menu slot="dropdown">
              <el-dropdown-item command="copy-plain-text">
                <span class="dropdown-text">复制纯文本</span>
              </el-dropdown-item>
              <el-dropdown-item disabled>
                <icon class="dropdown-icon" name="set-style" />
                <a href="/export/jpeg" target="_self" class="dropdown-text">自定义样式</a>
              </el-dropdown-item>
              <el-dropdown-item command="/export/ppt" divided>
                <icon class="dropdown-icon" name="preview" />
                <a href="/export/ppt" target="_self" class="dropdown-text">
                  {{ exportTextMap['/export/ppt'] }}
                </a>
              </el-dropdown-item>
              <el-dropdown-item command="/export/png" divided>
                <icon class="dropdown-icon" name="download" />
                <a href="/export/png" target="_self" class="dropdown-text">{{
                  exportTextMap['/export/png']
                }}</a>
              </el-dropdown-item>
              <el-dropdown-item command="/export/pdf">
                <icon class="dropdown-icon" name="download" />
                <a href="/export/pdf" target="_self" class="dropdown-text">
                  {{ exportTextMap['/export/pdf'] }}
                </a>
              </el-dropdown-item>
              <el-dropdown-item command="/export/html" disabled divided>
                <icon class="dropdown-icon" name="download" />
                <a href="/export/html" target="_self" class="dropdown-text">导出 HTML</a>
              </el-dropdown-item>
            </el-dropdown-menu>
          </el-dropdown>
          <span class="hint--bottom full-screen" @click="onFullScreenClick" aria-label="全屏">
            <icon class="header-icon" name="full-screen" />
          </span>
        </template>
      </nav>
    </h1>
  </header>
</template>

<script>
import 'hint.css'
import { exportTextMap } from '@config/constant'
import { createDocument, setActiveDocId, saveDocContent } from '@helper/storage'
import { trackEvent } from '@helper/analytics'

const MOBILE_BREAKPOINT = 1200
const detectMobileViewport = () => {
  const hasCoarsePointer =
    typeof window.matchMedia === 'function' && window.matchMedia('(pointer: coarse)').matches
  return window.innerWidth <= MOBILE_BREAKPOINT || hasCoarsePointer
}

export default {
  name: 'HeaderNav',

  props: {
    activeMode: {
      type: String,
      default: 'sv',
    },
    activePreviewMode: {
      type: String,
      default: 'both',
    },
    toolbarHidden: {
      type: Boolean,
      default: false,
    },
  },

  data() {
    return {
      isMobile: detectMobileViewport(),
      titleText: window.$appTitle,
      exportTextMap,
    }
  },

  mounted() {
    window.addEventListener('resize', this.onResize)
  },

  beforeDestroy() {
    window.removeEventListener('resize', this.onResize)
  },

  methods: {
    onResize() {
      this.isMobile = detectMobileViewport()
    },
    onModeSwitch(mode) {
      this.$emit('switch-mode', mode)
      trackEvent('header_switch_mode', 'header', mode)
    },
    onClearContent() {
      this.$emit('clear-content')
      trackEvent('header_clear_content', 'header', 'click')
    },
    launchFullScreen() {
      const element = document.getElementById('vditor')
      if (element.requestFullscreen) {
        element.requestFullscreen()
      } else if (element.msRequestFullscreen) {
        element.msRequestFullscreen()
      } else if (element.mozRequestFullScreen) {
        element.mozRequestFullScreen()
      } else if (element.webkitRequestFullscreen) {
        element.webkitRequestFullscreen(Element.ALLOW_KEYBOARD_INPUT)
      }
    },
    cancelFullScreen() {
      if (document.exitFullscreen) {
        document.exitFullscreen()
      } else if (document.msExitFullscreen) {
        document.msExitFullscreen()
      } else if (document.mozCancelFullScreen) {
        document.mozCancelFullScreen()
      } else if (document.webkitExitFullscreen) {
        document.webkitExitFullscreen()
      }
    },
    onThemeClick() {},
    emitPreviewAction(command) {
      const typeMap = {
        'copy-plain-text': 'copy-plain-text',
      }
      const type = typeMap[command]
      if (!type) return false
      this.$emit('preview-action', type)
      trackEvent('header_preview_action', 'header', type)
      return true
    },
    onFullScreenClick() {
      const isFullScreen =
        document.fullscreenElement ||
        document.mozFullScreenElement ||
        document.msFullscreenElement ||
        document.webkitFullscreenElement
      isFullScreen ? this.cancelFullScreen() : this.launchFullScreen()
      trackEvent('header_full_screen', 'header', isFullScreen ? 'exit' : 'enter')
    },
    handleCommand(command) {
      if (this.emitPreviewAction(command)) return
      this.$router.push(command)
      trackEvent('header_export', 'header', command)
    },
    handleMobileCommand(command) {
      if (this.emitPreviewAction(command)) return
      if (command === 'toggle-toolbar') {
        this.$emit('toggle-toolbar')
        trackEvent('header_mobile_command', 'header', command)
        return
      }
      if (command === 'import') {
        this.onImportClick()
        return
      }
      if (command === 'fullscreen') {
        this.onFullScreenClick()
        return
      }
      if (command.startsWith('/')) {
        this.$router.push(command)
        trackEvent('header_mobile_command', 'header', command)
      }
    },
    onImportClick() {
      trackEvent('header_import_click', 'header')
      const input = document.createElement('input')
      input.type = 'file'
      input.accept = '.md,.markdown,text/markdown'
      input.onchange = (e) => {
        const file = e.target.files[0]
        if (file) {
          const reader = new FileReader()
          reader.onload = (e) => {
            const content = e.target.result
            const title = (file.name || '').replace(/\.(md|markdown)$/i, '') || '导入的文档'
            const doc = createDocument(title)
            saveDocContent(doc.id, content)
            setActiveDocId(doc.id)
            this.$root.$emit('reload-content')
            trackEvent('header_import_success', 'header', title)
          }
          reader.readAsText(file)
        }
      }
      input.click()
    },
  },
}
</script>

<style lang="less">
@import './../../assets/styles/style.less';

[class*='hint--']:after {
  border-radius: 3px;
}

.el-popper[x-placement^='bottom'] {
  margin-top: 10px;
}

.el-dropdown .el-dropdown-link {
  height: @header-height;
  .flex-box-center(column);
}

.hint--bottom {
  cursor: pointer;
  pointer-events: all;
}

.el-dropdown-menu {
  margin: 0;

  .dropdown-icon {
    fill: @deep-black;
    vertical-align: middle;
    margin-right: 10px;
  }

  .dropdown-text {
    vertical-align: middle;
  }
}

.header-wrapper {
  position: fixed;
  top: 0;
  width: 100%;
  height: @header-height;
  line-height: @header-height;
  z-index: @hint-css-zindex;
  background-color: #fff;
  box-shadow: 0 0 12px 2px rgba(0, 0, 0, 0.1);
  transition: border 0.5s cubic-bezier(0.455, 0.03, 0.515, 0.955),
    background 0.5s cubic-bezier(0.455, 0.03, 0.515, 0.955);

  .header-area {
    width: 100%;
    height: 100%;
    padding: 0 2rem;
    max-width: @max-body-width;
    margin: auto;
    text-align: left;

    .header-link {
      display: inline-flex;
      height: @header-height;
      line-height: @header-height;

      .mark-markdown {
        width: @header-height;
        vertical-align: middle;
      }

      .header-text {
        margin-left: 10px;
        font-size: @font-medium;
        color: transparent;
        background-clip: text;
        background-image: linear-gradient(to right, #000000, #434343);
        vertical-align: middle;
      }
    }

    .button-group {
      float: right;
      display: flex;
      align-items: center;
      gap: 6px;

      .mode-switch {
        display: flex;
        align-items: center;
        gap: 4px;
        margin: 0 4px;

        .mode-switch__btn {
          height: 30px;
          line-height: 28px;
          padding: 0 8px;
          font-size: 12px;
          color: @deep-black;
          background: #fff;
          border: 1px solid @border-grey;
          border-radius: 6px;
          white-space: nowrap;
          cursor: pointer;
          transition: all 0.2s ease;
        }

        .mode-switch__btn--active {
          color: #fff;
          border-color: @brand;
          background: @brand;
        }

        .mode-switch__btn--danger {
          color: #d23f31;
          border-color: #f3b2ad;
          background: #fff7f7;
        }
      }

      .header-icon {
        margin: 0 10px;
        fill: @deep-black;
      }

      .full-screen {
        margin-right: -10px;
      }
    }
  }
}

@media (max-width: 1200px) {
  .header-wrapper {
    .header-area {
      display: flex;
      width: 100%;
      padding: 0 10px;
      align-items: center;
      justify-content: space-between;
      gap: 8px;

      .header-link {
        display: inline-flex;
        align-items: center;
      }

      .mark-markdown {
        width: 40px;
      }

      .button-group {
        float: none;
        display: flex;
        align-items: center;
        justify-content: flex-end;
        max-width: calc(100% - 50px);
        overflow-x: auto;
        overflow-y: hidden;
        white-space: nowrap;
        scrollbar-width: none;

        .mode-switch {
          max-width: calc(100vw - 120px);
          overflow-x: auto;
          overflow-y: hidden;
          padding-bottom: 2px;
          scrollbar-width: none;

          .mode-switch__btn {
            height: 28px;
            line-height: 26px;
            padding: 0 6px;
            min-width: 42px;
            font-size: 11px;
          }

          &::-webkit-scrollbar {
            display: none;
          }
        }

        &::-webkit-scrollbar {
          display: none;
        }

        .header-icon {
          margin: 0 6px;
        }
      }
    }
  }

  .hint--bottom::after {
    display: none;
  }
}
</style>
