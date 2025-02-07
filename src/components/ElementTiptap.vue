<template>
  <div
    v-if="editor"
    :style="elTiptapEditorStyle"
    :class="{ 'el-tiptap-editor--fullscreen': editorStateOptions.isFullscreen }"
    class="el-tiptap-editor"
  >
    <menu-bubble
      :editor="editor"
      :menuBubbleOptions="menuBubbleOptions"
    />

    <menu-bar
      v-if="showMenubar"
      :editor="editor"
      :class="{
        'border-top-radius': !isMenubarBottomPosition && showMenubar,
        'border-bottom-radius': isMenubarBottomPosition && !showFooter,
        'border-bottom': isMenubarBottomPosition && !showFooter,
        'order1': isMenubarBottomPosition,
        'el-tiptap-editor__menu-bar_divider_none': isMenubarBottomPosition,
      }"
    >
      <template
        v-if="$scopedSlots.menubar"
        v-slot="slotProps"
      >
        <slot
          name="menubar"
          v-bind="slotProps"
        />
      </template>
    </menu-bar>

    <div
      v-if="isCodeViewMode"
      :class="{
        'el-tiptap-editor__codemirror': true,
        'border-bottom-radius': isCodeViewMode && !isMenubarBottomPosition,
        'border-top-radius': isCodeViewMode && isMenubarBottomPosition,
      }"
    >
      <textarea ref="cmTextArea"></textarea>
    </div>

    <!-- use v-show to keep history -->
    <editor-content
      v-show="!isCodeViewMode"
      :editor="editor"
      :class="{
        'el-tiptap-editor__content': true,
        'el-tiptap-editor__content_border-bottom_none': showFooter || (showMenubar && isMenubarBottomPosition),
        'el-tiptap-editor__content_border-top_none': showMenubar && !isMenubarBottomPosition,
        'border-top-radius': !showMenubar || isMenubarBottomPosition,
        'border-bottom-radius': !showFooter && !isMenubarBottomPosition,
      }"
    />

    <slot
      name="footer"
      :editor="editor"
    >
      <div
        v-if="showFooter"
        :class="{
          'el-tiptap-editor__footer': true,
          'border-bottom-radius': showFooter,
          'order2': showMenubar && isMenubarBottomPosition,
        }"
      >
        <span class="el-tiptap-editor__characters">
          {{ t('editor.characters') }}: {{ characters }}
        </span>
      </div>
    </slot>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Watch, Provide, Model, Mixins, Vue } from 'vue-property-decorator';
import { Editor, EditorContent, Extension, EditorUpdateEvent } from 'tiptap';
import { Placeholder } from 'tiptap-extensions';
import { Node as ProsemirrorNode } from 'prosemirror-model';
import ContentAttributes from '@/extensions/content_attributes';
import Title from '@/extensions/title';
import { capitalize } from '@/utils/shared';
import { EVENTS } from '@/constants';
import EditorStylesMixin from '@/mixins/EditorStylesMixin';
import CodeViewMixin from '@/mixins/CodeViewMixin';
import i18nMixin from '@/mixins/i18nMixin';
import { EditorStateOptions } from '@/../types';

import MenuBar from './MenuBar/index.vue';
import MenuBubble from './MenuBubble/index.vue';

const COMMON_EMIT_EVENTS: EVENTS[] = [
  EVENTS.TRANSACTION,
  EVENTS.FOCUS,
  EVENTS.BLUR,
  EVENTS.PASTE,
  EVENTS.DROP,
];

@Component({
  components: {
    MenuBar,
    MenuBubble,
    EditorContent,
  },
  // fix @ProvideReactive
  // https://github.com/kaorun343/vue-property-decorator/issues/277#issuecomment-558594655
  inject: [],
})
export default class ElTiptap extends Mixins(EditorStylesMixin, CodeViewMixin, i18nMixin) {
  @Prop({
    type: Array,
    default: () => [],
  })
  readonly extensions!: any[];

  @Model('onUpdate', {
    type: String,
    default: '',
  })
  readonly content!: string;

  @Prop({
    type: String,
    default: '',
  })
  readonly placeholder!: string;

  @Prop({
    type: Object,
    default: () => ({}),
  })
  readonly editorProperties!: Object;

  @Prop({
    type: String,
    default: 'html',
    validator (output): boolean {
      return ['html', 'json'].includes(output);
    },
  })
  readonly output!: string;

  @Prop({
    type: Boolean,
    default: false,
  })
  readonly readonly!: boolean;

  @Prop({
    type: Boolean,
    default: undefined,
  })
  readonly spellcheck!: boolean | undefined;

  // TODO: popper.js
  @Prop({
    type: Object,
    default: () => ({}),
  })
  readonly menuBubbleOptions!: Object;

  editor: Editor | null = null;
  emitAfterOnUpdate: boolean = false;

  get characters (): number {
    if (!this.editor) return 0;

    return this.editor.state.doc.textContent.length;
  }

  get showFooter (): boolean {
    return this.charCounterCount && !this.isCodeViewMode;
  }

  get spellcheckEnabled (): boolean {
    return this.spellcheck == null
      ? this.$elementTiptapPlugin
        ? this.$elementTiptapPlugin.spellcheck
        : true
      : this.spellcheck;
  }

  @Watch('content')
  onContentChange (val: string): void {
    if (this.emitAfterOnUpdate) {
      this.emitAfterOnUpdate = false;
      return;
    }

    if (this.editor) this.editor.setContent(val);
  }

  @Watch('readonly')
  onReadonlyChange (): void {
    if (this.editor) {
      this.editor.setOptions({
        editable: !this.readonly,
      });
    }
  }

  private mounted () {
    const extensions = this.generateExtensions();

    const eventOptions = COMMON_EMIT_EVENTS.reduce((prev, event) => {
      return {
        ...prev,
        [`on${capitalize(event)}`]: () => this.emitEvent.bind(this)(event),
      };
    }, {});

    this.editor = new Editor({
      ...this.editorProperties,
      editable: !this.readonly,
      useBuiltInExtensions: false,
      extensions,
      ...eventOptions,
      content: this.content,
      onUpdate: this.onUpdate.bind(this),
    });

    this.$emit(this.genEvent(EVENTS.INIT), {
      editor: this.editor,
    });
  }

  private beforeDestroy () {
    if (this.editor) this.editor.destroy();
  }

  private generateExtensions (): Array<Extension> {
    const extensions: Extension[] = [...this.extensions];

    // spellcheck
    extensions.push(
      new ContentAttributes({
        spellcheck: this.spellcheckEnabled,
      }),
    );

    // placeholder
    extensions.push(this.initPlaceholderExtension());

    return extensions;
  }

  emitEvent (event: EVENTS) {
    this.$emit(this.genEvent(event), {
      editor: this.editor,
    });
  }

  onUpdate (options: EditorUpdateEvent) {
    this.emitAfterOnUpdate = true;

    let output;
    if (this.output === 'html') {
      output = options.getHTML();
    } else {
      output = JSON.stringify(options.getJSON());
    }

    this.$emit(this.genEvent(EVENTS.UPDATE), output, options);
  }

  @Provide() get editorStateOptions (): EditorStateOptions {
    return Vue.observable({
      isFullscreen: false,
      isCodeViewMode: false,
    });
  };

  private genEvent (event: EVENTS) {
    return `on${capitalize(event)}`;
  }

  private getTitleExtension (): Title | null {
    const doc = this.extensions.find(e => e.name === 'doc');
    if (doc.options.title) {
      const title = this.extensions.find(e => e.name === 'title');
      if (title) return title;
    }
    return null;
  }

  private initPlaceholderExtension (): Placeholder {
    const title = this.getTitleExtension();
    if (title) {
      // @ts-ignore
      return new Placeholder({
        emptyEditorClass: 'el-tiptap-editor--empty',
        emptyNodeClass: 'el-tiptap-editor__with-title-placeholder',
        showOnlyCurrent: false,
        // @ts-ignore
        emptyNodeText: (node: ProsemirrorNode) => {
          if (node.type.name === 'title') {
            return title.options.placeholder;
          }
          return this.placeholder;
        },
      });
    }
    return new Placeholder({
      emptyEditorClass: 'el-tiptap-editor--empty',
      emptyNodeClass: 'el-tiptap-editor__placeholder',
      emptyNodeText: this.placeholder,
    });
  }
};
</script>

<style lang="scss">
@import '../styles/editor.scss';
@import '../styles/command_button.scss';
</style>
