<template>
  <div class="quillWrapper">
    <slot name="toolbar" />
    <div
      :id="id"
      ref="quillContainer" />
    <input
      v-if="useCustomImageHandler"
      id="file-upload"
      ref="fileInput"
      type="file"
      accept="image/*"
      style="display:none;"
      @change="emitImageInfo($event)">
  </div>
</template>

<script>
import _Quill from 'quill'
import { merge } from 'lodash'
import defaultToolbar from './helpers/default-toolbar'
import oldApi from './helpers/old-api'
import MarkdownShortcuts from './helpers/markdown-shortcuts'

export const Quill = _Quill
export default {
  name: 'VueEditor',
  mixins: [oldApi],
  props: {
    id: {
      type: String,
      default: 'quill-container',
    },
    placeholder: {
      type: String,
      default: '',
    },
    value: {
      type: String,
      default: '',
    },
    disabled: {
      type: Boolean,
    },
    editorToolbar: Array,
    editorOptions: {
      type: Object,
      required: false,
      default: () => ({}),
    },
    useCustomImageHandler: {
      type: Boolean,
      default: false,
    },
    useMarkdownShortcuts: {
      type: Boolean,
      default: false,
    },
  },

  data: () => ({
    quill: null,
  }),

  watch: {
    value(val) {
      if (val != this.quill.root.innerHTML && !this.quill.hasFocus()) {
        this.quill.root.innerHTML = val
      }
    },
    disabled(status) {
      this.quill.enable(!status)
    },
  },

  mounted() {
    this.registerCustomModules(Quill)
    this.registerPrototypes()
    this.initializeEditor()
  },

  beforeDestroy() {
    this.quill = null
    delete this.quill
  },

  methods: {
    initializeEditor() {
      this.setupQuillEditor()
      this.checkForCustomImageHandler()
      this.handleInitialContent()
      this.registerEditorEventListeners()
      this.$emit('ready', this.quill)
    },

    setupQuillEditor() {
      const editorConfig = {
        debug: false,
        modules: this.setModules(),
        theme: 'snow',
        placeholder: this.placeholder ? this.placeholder : '',
        readOnly: this.disabled ? this.disabled : false,
      }

      this.prepareEditorConfig(editorConfig)
      this.quill = new Quill(this.$refs.quillContainer, editorConfig)
    },

    setModules() {
      const modules = {
        toolbar: this.editorToolbar ? this.editorToolbar : defaultToolbar,
      }
      if (this.useMarkdownShortcuts) {
        Quill.register('modules/markdownShortcuts', MarkdownShortcuts, true)
        modules.markdownShortcuts = {}
      }
      return modules
    },

    prepareEditorConfig(editorConfig) {
      if (
        Object.keys(this.editorOptions).length > 0
        && this.editorOptions.constructor === Object
      ) {
        if (
          this.editorOptions.modules
          && typeof this.editorOptions.modules.toolbar !== 'undefined'
        ) {
          // We don't want to merge default toolbar with provided toolbar.
          delete editorConfig.modules.toolbar
        }
        merge(editorConfig, this.editorOptions)
      }
    },

    registerPrototypes() {
      Quill.prototype.getHTML = function () {
        return this.container.querySelector('.ql-editor').innerHTML
      }
      Quill.prototype.getWordCount = function () {
        return this.container.querySelector('.ql-editor').innerText.length
      }
    },

    registerEditorEventListeners() {
      this.quill.on('text-change', this.handleTextChange)
      this.quill.on('selection-change', this.handleSelectionChange)
      this.listenForEditorEvent('text-change')
      this.listenForEditorEvent('selection-change')
      this.listenForEditorEvent('editor-change')
    },

    listenForEditorEvent(type) {
      this.quill.on(type, (...args) => {
        this.$emit(type, ...args)
      })
    },

    handleInitialContent() {
      if (this.value) this.quill.root.innerHTML = this.value // Set initial editor content
    },

    handleSelectionChange(range, oldRange) {
      if (!range && oldRange) this.$emit('blur', this.quill)
      else if (range && !oldRange) this.$emit('focus', this.quill)
    },

    handleTextChange() {
      const editorContent = this.quill.getHTML() === '<p><br></p>' ? '' : this.quill.getHTML()
      this.$emit('input', editorContent)
    },

    checkForCustomImageHandler() {
      this.useCustomImageHandler === true ? this.setupCustomImageHandler() : ''
    },

    setupCustomImageHandler() {
      const toolbar = this.quill.getModule('toolbar')
      toolbar.addHandler('image', this.customImageHandler)
    },

    customImageHandler(image, callback) {
      this.$refs.fileInput.click()
    },

    emitImageInfo($event) {
      const resetUploader = function () {
        const uploader = document.getElementById('file-upload')
        uploader.value = ''
      }
      const file = $event.target.files[0]
      const Editor = this.quill
      const range = Editor.getSelection()
      const cursorLocation = range.index
      this.$emit('imageAdded', file, Editor, cursorLocation, resetUploader)
    },
  },
}
</script>

<style src="quill/dist/quill.snow.css">
</style>
<style src="./styles/vue2-editor.scss" lang='scss'>
</style>
