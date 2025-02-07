<template>
  <el-dropdown
    placement="bottom"
    trigger="click"
    @command="lineHeight => editorContext.commands.line_height({ lineHeight })"
  >
    <command-button
      :tooltip="t('editor.extensions.LineHeight.tooltip')"
      :readonly="editorStateOptions.isCodeViewMode"
      icon="text-height"
    />
    <el-dropdown-menu
      slot="dropdown"
      class="el-tiptap-dropdown-menu"
    >
      <el-dropdown-item
        v-for="lineHeight in lineHeights"
        :key="lineHeight"
        :command="lineHeight"
        :class="{
          'el-tiptap-dropdown-menu__item--active': isLineHeightActive(lineHeight),
        }"
        class="el-tiptap-dropdown-menu__item"
      >
        <span>{{ lineHeight }}</span>
      </el-dropdown-item>
    </el-dropdown-menu>
  </el-dropdown>
</template>

<script lang="ts">
import { Component, Prop, Mixins, Inject } from 'vue-property-decorator';
import { MenuData } from 'tiptap';
import { EditorStateOptions } from '@/../types';
import { Dropdown, DropdownMenu, DropdownItem } from 'element-ui';
import i18nMixin from '@/mixins/i18nMixin';
import { isLineHeightActive } from '@/utils/line_height';
import CommandButton from './CommandButton.vue';

@Component({
  components: {
    [Dropdown.name]: Dropdown,
    [DropdownMenu.name]: DropdownMenu,
    [DropdownItem.name]: DropdownItem,
    CommandButton,
  },
})
export default class LineHeightDropdown extends Mixins(i18nMixin) {
  @Prop({
    type: Object,
    required: true,
  })
  readonly editorContext!: MenuData;

  @Inject() readonly editorStateOptions!: EditorStateOptions;

  private get editor () {
    return this.editorContext.editor;
  }

  private get lineHeights () {
    return this.editor.extensions.options.line_height.lineHeights;
  }

  private isLineHeightActive (lineHeight: string): boolean {
    return isLineHeightActive(this.editor.state, lineHeight);
  }
};
</script>
