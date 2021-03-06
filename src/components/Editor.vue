<template>
  <div :id="`monaco-${id}`" class="editor-view"></div>
</template>


<script lang="ts">
import { Component, Vue, Prop, Model, Watch } from 'vue-property-decorator'
import { Getter } from 'vuex-class'
import * as monaco from 'monaco-editor/esm/vs/editor/editor.api'
import EditorDark from '@/assets/scss/theme/editor-dark.json'
import EditorNight from '@/assets/scss/theme/editor-night.json'

@Component
export default class Editor extends Vue {
  @Prop({ required: true }) public id!: string
  @Prop({ required: true }) public lang!: string
  @Prop({ default: 14 }) public fontSize!: number
  @Prop({ default: 'hidden' }) public scrollbarStatus: 'auto' | 'visible' | 'hidden' | undefined

  @Model('change', { type: String }) private readonly value!: string

  @Getter('currentTheme') private theme!: Theme

  private editor: monaco.editor.IStandaloneCodeEditor | null = null

  @Watch('value')
  private handleValueChanged(val: string) {
    if (this.editor) {
      if (val !== this.editor.getValue()) {
        this.editor.setValue(val)
      }
    }
  }

  @Watch('lang')
  private handleLangChanged(val: string) {
    if (this.editor) {
      this.editor.dispose()
      this.initEditor()
    }
  }

  private initEditor(): void | boolean {
    const defaultOptions: monaco.editor.IStandaloneEditorConstructionOptions = {
      value: this.value,
      language: this.lang,
      readOnly: false,
      fontSize: this.fontSize,
      scrollBeyondLastLine: false,
      lineNumbers: 'off',
      renderLineHighlight: 'none',
      matchBrackets: 'near',
      folding: false,
      theme: this.getTheme(),
      lightbulb: {
        enabled: false,
      },
      minimap: {
        enabled: false,
      },
      scrollbar: {
        horizontal: this.scrollbarStatus,
        vertical: this.scrollbarStatus,
        useShadows: false,
        alwaysConsumeMouseWheel: false,
      },
    }
    // Create
    const id = document.getElementById(`monaco-${this.id}`)
    if (!id) {
      return false
    }
    this.editor = monaco.editor.create(id, defaultOptions)
    // event changed
    this.editor.onDidChangeModelContent((event) => {
      if (this.editor) {
        const value = this.editor.getValue()
        if (value !== this.value) {
          this.$emit('input', value, event)
          this.$emit('change', value, event)
        }
      }
    })
    // Command method
    // tslint:disable-next-line:no-bitwise
    this.editor.addCommand(monaco.KeyMod.CtrlCmd | monaco.KeyCode.Enter, () => {
      this.$emit('enter-event', this.value)
    })
    // Update editor options
    const model = this.editor.getModel()
    if (model) {
      model.updateOptions({ tabSize: 2 })
    }
    this.editor.onDidFocusEditorText(() => {
      this.$emit('focus')
    })
    this.editor.onDidBlurEditorText(() => {
      this.$emit('blur')
    })
  }
  private editorLayout() {
    if (this.editor) {
      this.editor.layout()
    }
  }
  private defineTheme() {
    const dark = EditorDark as monaco.editor.IStandaloneThemeData
    const night = EditorNight as monaco.editor.IStandaloneThemeData
    monaco.editor.defineTheme('editor-dark', dark)
    monaco.editor.defineTheme('editor-night', night)
  }
  private getTheme(): string {
    if (this.theme === 'dark') {
      return 'editor-dark'
    } else if (this.theme === 'night') {
      return 'editor-night'
    } else {
      return 'vs'
    }
  }

  private mounted() {
    this.initEditor()
  }
  private created() {
    this.defineTheme()
    window.onresize = () => {
      this.editorLayout()
    }
  }
  private beforeDestroy() {
    if (this.editor) {
      const model = this.editor.getModel()
      if (model) {
        model.dispose()
      }
      this.editor.dispose()
      this.editor = null
    }
  }
}
</script>


<style lang="scss">
.editor-view {
  height: 100%;
  width: 100%;
  position: relative;
}
.decorationsOverviewRuler {
  display: none;
}
</style>
