<template>
  <div
    class="side-bar-folder"
  >
  <div class="drop-zone"
  @drop="onDrop($event, 1)"
      @dragover.prevent
      @dragenter.prevent>
  </div>
    <div
      class="folder-name" @click="folderNameClick(folder)"
      :style="{'padding-left': `${(depth * 20) + 20}px`}"
      :class="[{ 'active': folder.id === activeItem.id }]"
      :title="folder.pathname"
      ref="folder"
      v-show="!isHidden"
      draggable
      @dragstart="startDrag($event, folder)"
      >
      <svg class="icon" aria-hidden="true">
        <use :xlink:href="`#${folder.isCollapsed ? 'icon-folder-close' : 'icon-folder-open'}`"></use>
      </svg>
      <input
        type="text"
        @click.stop="noop"
        class="rename"
        v-if="renameCache === folder.pathname"
        v-model="newName"
        ref="renameInput"
        @keydown.enter="rename"
      >
      <span v-else class="text-overflow">{{folder.name}}</span>
    </div>
    <!-- <div
      class="folder-contents"
      v-if="!folder.isCollapsed"
    >
      <folder
        v-for="(childFolder, index) of folder.folders" :key="index + 'folder'"
        :folder="childFolder"
        :depth="depth + 1"
      ></folder>
      <input
        type="text" v-if="createCache.dirname === folder.pathname"
        class="new-input"
        :style="{'margin-left': `${depth * 5 + 15}px` }"
        ref="input"
        @keydown.enter="handleInputEnter"
        v-model="createName"
      >
      <draggable v-model="list" @end="onDrop">
      <file
        v-for="(file, index) of folder.files" :key="index + 'file'"
        :file="file"
        :depth="depth + 1"
      ></file>
    </draggable>
    </div> -->
  </div>
</template>

<script>
import { mapState } from 'vuex'
import { showContextMenu } from '../../contextMenu/sideBar'
import bus from '../../bus'
import { createFileOrDirectoryMixins } from '../../mixins'
let nodeConsole = require('console')
let myConsole = new nodeConsole.Console(process.stdout, process.stderr)

export default {
  mixins: [createFileOrDirectoryMixins],
  name: 'folder',
  data () {
    return {
      createName: '',
      newName: '',
      isHidden: Boolean(this.depth),
      id: this.folder.id
    }
  },
  props: {
    folder: {
      type: Object,
      required: true
    },
    depth: {
      type: Number,
      required: true
    },
    binderTree: {
      type: Object,
      required: true
    }
  },
  components: {
    File: () => import('./treeFile.vue')
  },
  computed: {
    ...mapState({
      renameCache: state => state.project.renameCache,
      createCache: state => state.project.createCache,
      activeItem: state => state.project.activeItem,
      clipboard: state => state.project.clipboard
    })
  },
  created () {
    this.$nextTick(() => {
      this.$refs.folder.addEventListener('contextmenu', event => {
        event.preventDefault()
        this.$store.dispatch('CHANGE_ACTIVE_ITEM', this.folder)
        showContextMenu(event, !!this.clipboard)
      })
      bus.$on('SIDEBAR::show-new-input', this.handleInputFocus)
      bus.$on('SIDEBAR::show-rename-input', this.focusRenameInput)
    })
  },
  methods: {
    folderNameClick (folder) {
      if (folder.isCollapsed) {
        const toShow = []
        const allItems = [].concat(folder.folders).concat(folder.files)
        allItems.forEach((item) => toShow.push(item.id))
        this.$emit('show-folder', toShow)
        folder.isCollapsed = false
      } else {
        const toHide = []
        const prefix = folder.name.substring(0, folder.name.indexOf('#')) + '.'
        this.binderTree.forEach((item) => {
          if (item.name.startsWith(prefix)) {
            toHide.push(item.id)
            if (item.isDirectory) { item.isCollapsed = true }
          }
        })
        this.$emit('hide-folder', toHide)
        folder.isCollapsed = true
      }
    },
    noop () {},
    focusRenameInput () {
      this.$nextTick(() => {
        if (this.$refs.renameInput) {
          this.$refs.renameInput.focus()
          this.newName = this.folder.name
        }
      })
    },
    rename () {
      const { newName } = this
      if (newName) {
        this.$store.dispatch('RENAME_IN_SIDEBAR', newName)
      }
    },
    startDrag (evt, item) {
      evt.dataTransfer.dropEffect = 'move'
      evt.dataTransfer.effectAllowed = 'move'
      evt.dataTransfer.setData('itemID', item.id)
    },
    onDrop (evt, list) {
      myConsole.log('dropped')
      const itemID = evt.dataTransfer.getData('itemID')
      const item = this.items.find((item) => item.id === itemID)
      item.list = list
    }
  }
}
</script>

<style scoped>
  .side-bar-folder {
    & > .folder-name {
      cursor: default;
      user-select: none;
      display: flex;
      align-items: center;
      height: 30px;
      padding-right: 15px;
      & > svg {
        flex-shrink: 0;
        color: var(--sideBarIconColor);
        margin-right: 5px;
      }
      &:hover {
        background: var(--sideBarItemHoverBgColor);
      }
    }
  }
  .new-input, input.rename {
    outline: none;
    height: 22px;
    margin: 5px 0;
    padding: 0 6px;
    color: var(--sideBarColor);
    border: 1px solid var(--floatBorderColor);
    background: var(--floatBorderColor);
    width: 70%;
    border-radius: 3px;
  }
  .drop-zone {
    margin-bottom: 10px;
    padding: 10px
  }
</style>
