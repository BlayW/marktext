<template>
  <div
    :title="file.pathname"
    class="side-bar-file"
    :style="{'padding-left': `${(depth * 20) + 20}px`, 'opacity': file.isMarkdown ? 1 : 0.75 }"
    @click="handleFileClick()"
    :class="[{'current': currentFile.pathname === file.pathname, 'active': file.id === activeItem.id }]"
    ref="file"
    v-show="!isHidden"
    draggable
    @dragstart="startDrag($event, file)"
    @drop="onDrop($event)"
    >
    <file-icon
      :name="file.name"
    ></file-icon>
    <input
      type="text"
      @click.stop="noop"
      class="rename"
      v-if="renameCache === file.pathname"
      v-model="newName"
      ref="renameInput"
      @keydown.enter="rename"
    >
    <span v-else>{{ file.name }}</span>
  </div>
</template>

<script>
import FileIcon from './icon.vue'
import { mapState } from 'vuex'
import { fileMixins } from '../../mixins'
import { showContextMenu } from '../../contextMenu/sideBar'
import bus from '../../bus'
// import draggable from 'vuedraggable'
let nodeConsole = require('console')
let myConsole = new nodeConsole.Console(process.stdout, process.stderr)

export default {
  mixins: [fileMixins],
  name: 'file',
  data () {
    return {
      newName: '',
      isHidden: Boolean(this.depth),
      id: this.file.id
    }
  },
  props: {
    file: {
      type: Object,
      required: true
    },
    depth: {
      type: Number,
      required: true
    }
  },
  components: {
    FileIcon
    // draggable
  },
  computed: {
    ...mapState({
      renameCache: state => state.project.renameCache,
      activeItem: state => state.project.activeItem,
      clipboard: state => state.project.clipboard,
      currentFile: state => state.editor.currentFile,
      tabs: state => state.editor.tabs
    })
  },
  created () {
    this.$nextTick(() => {
      this.$refs.file.addEventListener('contextmenu', event => {
        event.preventDefault()
        this.$store.dispatch('CHANGE_ACTIVE_ITEM', this.file)
        showContextMenu(event, !!this.clipboard)
      })

      bus.$on('SIDEBAR::show-rename-input', this.focusRenameInput)
    })
  },
  methods: {
    noop () {},
    focusRenameInput () {
      this.$nextTick(() => {
        if (this.$refs.renameInput) {
          this.$refs.renameInput.focus()
          this.newName = this.file.name
        }
      })
    },
    rename () {
      const { newName } = this
      if (newName) {
        this.$store.dispatch('RENAME_IN_SIDEBAR', newName)
      }
    },
    onEnd (event) {
      console.log('Reordered list: ', this.list)
    },
    startDrag (evt, item) {
      evt.dataTransfer.dropEffect = 'move'
      evt.dataTransfer.effectAllowed = 'move'
      evt.dataTransfer.setData('draggedItem', item)
      myConsole.log(item.name)
    },
    onDrop (evt) {
      myConsole.log('triggered')
      // const draggedItem = evt.dataTransfer.getData('draggedItem')
      // myConsole.log('draggedItem: ', draggedItem, '\n dropped on: ', list)
    }
  }
}
</script>

<style scoped>
  .side-bar-file {
    display: flex;
    position: relative;
    align-items: center;
    cursor: default;
    user-select: none;
    height: 30px;
    box-sizing: border-box;
    padding-right: 15px;
    &:hover {
      background: var(--sideBarItemHoverBgColor);
    }
    & > span {
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    &::before {
      content: '';
      position: absolute;
      display: block;
      left: 0;
      background: var(--themeColor);
      width: 2px;
      height: 0;
      top: 50%;
      transform: translateY(-50%);
      transition: all .2s ease;
    }
  }
  .side-bar-file.current::before {
    height: 100%;
  }
  .side-bar-file.current > span {
    color: var(--themeColor);
  }
  .side-bar-file.active > span {
    color: var(--sideBarTitleColor);
  }
  input.rename {
    height: 22px;
    outline: none;
    margin: 5px 0;
    padding: 0 8px;
    color: var(--sideBarColor);
    border: 1px solid var(--floatBorderColor);
    background: var(--floatBorderColor);
    width: 100%;
    border-radius: 3px;
  }
</style>
