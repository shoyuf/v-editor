<template>
  <div class="v-editor">
    <ckeditor
      :editor="ClassicEditor"
      :value="value"
      :disabled="disabled"
      :config="editorConfig"
      @input="onInput"
      @ready="onReady"
      v-on="$listeners"
    />
    <div
      class="toggle-full-screen"
      :class="{'is-full-screen': isFullScreen}"
      @click="toggleFullScreen"
    >
      <full-screen-exit-icon v-if="isFullScreen" />
      <full-screen-icon v-else />
    </div>

    <upload-to-ali
      v-show="false"
      ref="uploadToAli"
      value
      v-bind="uploadOptions"
      :accept="uploaderAccept"
    />
    <img-preview v-model="previewImageUrl" />
  </div>
</template>

<script>
import ClassicEditor from '@ckeditor/ckeditor5-editor-classic/src/classiceditor'
import UploadToAli from '@femessage/upload-to-ali'
import ImgPreview from '@femessage/img-preview'

import defaultEditorOptions from './defaultEditorOptions'
import debounce from 'lodash-es/debounce'
import Uploader from './plugin/Uploader'
import CKEditor from '@ckeditor/ckeditor5-vue'

import fullScreenIcon from './assets/fullscreen.vue'
import fullScreenExitIcon from './assets/fullscreenexit.vue'

import ImagePreview from './plugin/ImagePreview'
import './translations'

const ROW_HEIGHT = 24
const INNER_PADDING = 8 * 2 + 1 * 2 // .ck-content padding + border

export default {
  name: 'VEditor',
  components: {
    UploadToAli,
    ckeditor: CKEditor.component,
    fullScreenIcon,
    fullScreenExitIcon,
    ImgPreview
  },
  props: {
    /**
     * 编辑区占位符
     */
    placeholder: {
      type: String,
      default: ''
    },
    /**
     * @deprecated
     * 编辑区高度（不包括 toolbar），支持数字类型（默认单位 px）和 css 长度字符串
     * 如果设置了 autosize，该选项将失效
     */
    height: {
      type: [Number, String],
      default: undefined
    },
    /**
     * upload-to-ali的参数，参考[upload-to-ali文档](https://femessage.github.io/upload-to-ali)
     */
    uploadOptions: {
      type: Object,
      default: () => ({
        compressOptions: {
          maxWidth: Infinity
        }
      })
    },
    /**
     * 编辑的内容，返回一段HTML，支持v-model。
     * 如果需要保持编辑器内和阅读区域样式一致，在使用的标签上添加 v-editor 和 markdown-body 两个 class 即可
     */
    value: {
      type: String,
      default: ''
    },
    /**
     * editor配置
     *
     */
    editorOptions: {
      type: Object,
      default: () => ({})
    },
    /**
     * 编辑器是否可编辑
     */
    disabled: {
      type: Boolean,
      default: false
    },
    /**
     * Upload Fail事件处理，入参status为boolean，true表示upload失败原因是不满足uploadOptions导致的；false表示upload被catch错误了，此时会传入第二个参数error信息
     */
    onUploadFail: {
      type: Function,
      default() {
        alert('上传失败，请重试')
      }
    },
    /**
     * 自适应内容高度，传入对象，如，{ minRows: 2, maxRows: 6 }
     */
    autosize: {
      type: Object,
      default: () => ({})
    }
  },
  data() {
    return {
      editor: null,
      ClassicEditor,
      isFullScreen: false,
      uploaderAccept: '*/*',
      previewImageUrl: ''
    }
  },
  computed: {
    editorConfig() {
      // $refs 在 mounted 阶段才挂载，这里不能直接传实例
      return Object.assign(
        {},
        defaultEditorOptions,
        {
          placeholder: this.placeholder,
          extraPlugins: [
            Uploader(this.uploadFile),
            ImagePreview(this.imagePreview)
          ],
          autosave: {
            save: debounce(editor => {
              /**
               * 建议自动保存事件，当 8 秒内未触发 input 事件时触发；
               * 另外，v-editor 还支持 focus、blur 等 ckeditor-vue 事件；
               * 见[文档](https://ckeditor.com/docs/ckeditor5/latest/builds/guides/integration/frameworks/vuejs.html#component-events)
               *
               * @property {string} data - 当前内容
               */
              this.$emit('autosave', editor.getData())
            }, 8000)
          },
          language: 'zh-cn'
        },
        this.editorOptions
      )
    },
    autosizeIsEmpty() {
      return !Object.keys(this.autosize).length
    }
  },
  watch: {
    height: 'setHeight',
    autosize: {
      handler: 'resizeHeight',
      deep: true
    }
  },
  methods: {
    resizeHeight() {
      if (this.autosizeIsEmpty || !this.editor) {
        return
      }
      let {minRows, maxRows} = this.autosize
      const {element} = this.editor.ui.view
      const main = element.querySelector('.ck-editor__main')
      const content = element.querySelector('.ck-content')
      const result = {}
      if (!minRows) {
        minRows = 3
      }
      const minHeight = ROW_HEIGHT * minRows + INNER_PADDING
      const maxHeight = ROW_HEIGHT * maxRows + INNER_PADDING || Infinity
      content.style.minHeight = `${minHeight}px`
      const resize = () => {
        this.$nextTick(() => {
          let height = content.scrollHeight
          if (height <= minHeight) {
            result.height = `${minHeight}px`
          }
          if (height > minHeight && height < maxHeight) {
            result.height = ''
          }
          if (height >= maxHeight) {
            result.height = `${maxHeight}px`
          }
          content.style.minHeight = `${minHeight}px`
          main.style.height = result.height
        })
      }
      resize()
      this.editor.model.document.on('change:data', resize)
    },
    imagePreview(url) {
      this.previewImageUrl = url
    },
    setHeight() {
      if (!this.height || !this.editor || !this.autosizeIsEmpty) return
      let {height} = this
      if (!isNaN(+height)) height += 'px'
      const {element} = this.editor.ui.view
      const main = element.querySelector('.ck-editor__main')
      main.style.height = height
    },
    onInput(content) {
      this.$emit('input', content)
    },
    onReady(editor) {
      this.editor = editor
      editor.ui.view.element
        .querySelector('.ck-editor__main')
        .classList.add('markdown-body')
      this.setHeight()
      this.resizeHeight()
    },
    /**
     * @param {File} file 选择的文件
     */
    async uploadFile(file) {
      this.$emit('upload-start')
      // 模拟 upload-to-ali 的 upload传参
      try {
        const res = await this.$refs.uploadToAli.upload({
          target: {files: [file]}
        })
        // res没有返回意味着上传过程中发现upload文件大小超出限制或其他不能上传的限制导致上传不能执行
        if (res) {
          this.$emit('upload-end', true, res)
          return res
        } else {
          this.$emit('upload-end', false, 'fail')
          this.onUploadFail(true)
        }
      } catch (e) {
        this.$emit('upload-end', false, e)
        this.onUploadFail(false, e)
      }
    },
    toggleFullScreen() {
      this.isFullScreen = !this.isFullScreen
      this.editor.ui.view.element.classList.toggle('full-screen')
    }
  }
}
</script>

<style lang="less" scoped>
@ck-border-color: #dcdee6;
@button-size: 24px;
@icon-size: 16px;
@font-size: 12px;
@scrollbar-color: #c6c7ca;
@scrollbar-size: 4px;
@ck-header-label-width: 45px;
@full-screen-index: 10000;

.toggle-full-screen {
  position: absolute;
  width: @icon-size;
  height: @icon-size;
  right: 8px;
  top: 48px;
  cursor: pointer;

  &.is-full-screen {
    position: fixed;
    z-index: @full-screen-index;
  }

  > svg {
    width: 100%;
    height: 100%;
  }
}

.v-editor {
  position: relative;
  min-width: 400px;

  &::v-deep {
    .full-screen {
      position: fixed;
      top: 0;
      right: 0;
      left: 0;
      z-index: @full-screen-index;

      .ck-editor__main {
        height: calc(100vh - 41px) !important;
      }
    }

    .ck {
      &.ck-editor__top {
        .ck-heading-dropdown,
        .ck-font-size-dropdown {
          .ck-button .ck-button__label {
            width: @ck-header-label-width;
            text-align: center;
            font-size: @font-size;
            line-height: @button-size;
            height: @button-size;
          }
        }
      }

      &.ck-editor__main {
        /* 控制整个滚动条 */
        ::-webkit-scrollbar {
          width: @scrollbar-size;
          height: @scrollbar-size;
        }

        /* 滚动条两端方向按钮 */
        ::-webkit-scrollbar-button {
          display: none;
        }

        /* 滚动条中间滑动部分 */
        ::-webkit-scrollbar-thumb {
          background-color: @scrollbar-color;
          border-radius: @scrollbar-size / 2;
        }

        /* 滚动条右下角区域 */
        ::-webkit-scrollbar-corner {
          display: none;
        }

        .ck-content {
          // use for height prop
          height: 100%;

          &.ck-editor__editable {
            border-color: @ck-border-color;

            &.ck-editor__editable_inline > :first-child {
              margin-top: 8px;
            }

            &.ck-editor__editable_inline > :last-child {
              margin-bottom: 8px;
            }

            &:not(.ck-editor__nested-editable).ck-focused {
              box-shadow: none;
            }

            .ck-placeholder::before {
              color: #c0c4cc;
            }
          }
        }
      }
    }
  }
}
</style>
<style lang="less" src="./assets/richtext.less"></style>
<style lang="less">
@ck-button-hover-background-color: #f0f2f5;
@ck-border-color: #dcdee6;
@toolbar-height: 36px;
@button-size: 24px;
@icon-size: 16px;
@button-distance: 4px;

// <body> 子级同时有的元素
.ck {
  &.ck-balloon-panel {
    z-index: 3000;
  }

  .ck-button {
    margin: 0;
    padding: 0;
    min-width: unset;
    min-height: unset;
    cursor: pointer;
    // margin: 12px 0;
    .ck.ck-icon {
      width: @icon-size;
      height: @icon-size;
    }

    &:not(.ck-disabled):hover {
      background: @ck-button-hover-background-color;
    }
  }

  .ck-toolbar {
    border-color: @ck-border-color;
    background: #fff;

    .ck.ck-toolbar__separator {
      height: @icon-size;
      margin: auto @button-distance*2;
    }

    .ck-toolbar__items {
      height: @toolbar-height;

      > .ck:not(.ck-toolbar__separator),
      > .ck-file-dialog-button,
      > .ck-button_with-text {
        padding: 0;
        margin: 0;
      }

      .ck-button {
        width: @button-size;
        height: @button-size;
        line-height: @button-size;
      }

      .ck-list__item {
        .ck-button {
          width: 100%;
          height: auto;
          line-height: 1;
        }
      }

      .ck-dropdown {
        &.ck-heading-dropdown {
          .ck-dropdown__button {
            padding: 0;
          }
        }

        .ck-button.ck-dropdown__button {
          width: 100%;
        }

        .ck-dropdown__arrow {
          margin: 0;
          right: 0;
        }
      }

      .ck-color-table {
        .ck-color-table__remove-color {
          width: 100%;
          padding-left: 8px;
        }
      }
    }
  }
}
</style>
