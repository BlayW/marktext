<template>
  <div class="tree-view">
    <div class="title">
    </div>

    <!-- Opened tabs -->
    <div class="opened-files">
      <div class="title">
        <svg class="icon icon-arrow" :class="{'fold': !showOpenedFiles}" aria-hidden="true" @click.stop="toggleOpenedFiles()">
          <use xlink:href="#icon-arrow"></use>
        </svg>
        <span class="default-cursor text-overflow" @click.stop="toggleOpenedFiles()">Opened files</span>
        <a href="javascript:;" @click.stop="saveAll(false)" title="Save All">
          <svg class="icon" aria-hidden="true">
            <use xlink:href="#icon-save-all"></use>
          </svg>
        </a>
        <a href="javascript:;" @click.stop="saveAll(true)" title="Close All">
          <svg class="icon" aria-hidden="true">
            <use xlink:href="#icon-close-all"></use>
          </svg>
        </a>
      </div>
      <div class="opened-files-list" v-show="showOpenedFiles">
      </div>
    </div>

    <!-- Project tree view -->
    <div
      class="project-tree" v-if="projectTree && !isBinder"
    >
      <div class="title">
        <svg class="icon icon-arrow" :class="{'fold': !showDirectories}" aria-hidden="true" @click.stop="toggleDirectories()">
          <use xlink:href="#icon-arrow"></use>
        </svg>
        <span class="default-cursor text-overflow" @click.stop="toggleDirectories()">{{ projectTree.name }}</span>
        <a href="javascript:;" @click.stop="makeBinder()" title="Make Binder">
          <svg class="icon" aria-hidden="true">
            <use xlink:href="#icon-list"></use>
          </svg>
        </a>
      </div>
      <div class="tree-wrapper" v-show="showDirectories">
        <draggable v-model="list" @end="onDrop">
        <folder
          v-for="(folder, index) of projectTree.folders" :key="index + 'folder'"
          :folder="folder"
          :depth="depth"
        ></folder>
        <input
          type="text" class="new-input" v-show="createCache.dirname === projectTree.pathname"
          :style="{'margin-left': `${depth * 5 + 15}px` }"
          ref="input"
          v-model="createName"
          @keydown.enter="handleInputEnter"
        >
        <file
          v-for="(file, index) of projectTree.files" :key="index + 'file'"
          :file="file"
          :depth="depth"
        ></file>
      </draggable>
        <div class="empty-project" v-if="projectTree.files.length === 0 && projectTree.folders.length === 0">
          <span>Empty project</span>
          <a href="javascript:;" @click.stop="createFile">Create File</a>
        </div>
      </div>
    </div>
    <!-- Binder tree view -->
    <div
      class="project-tree" v-if="projectTree && isBinder"
    >
      <div class="title">
        <svg class="icon icon-arrow" :class="{'fold': !showDirectories}" aria-hidden="true" @click.stop="toggleDirectories()">
          <use xlink:href="#icon-arrow"></use>
        </svg>
        <span class="default-cursor text-overflow" @click.stop="toggleDirectories()">{{ projectTree.name }}</span>
      </div>
      <div class="tree-wrapper" v-show="showDirectories">
        <draggable v-model="list" @end="onDrop">
          <div v-for="item in projectTree.allItems">
          <div v-if="item.isDirectory">
          <folder
          :folder="item"
          :depth="depth"
        ></folder>
        <input
          type="text" class="new-input" v-show="createCache.dirname === projectTree.pathname"
          :style="{'margin-left': `${depth * 5 + 15}px` }"
          ref="input"
          v-model="createName"
          @keydown.enter="handleInputEnter"
        >
          </div>
          <div v-else-if="item.isFile">
          <file
          :file="item"
          :depth="depth"
        ></file>
          </div>
          </div>
      </draggable>
        <div class="empty-project" v-if="projectTree.files.length === 0 && projectTree.folders.length === 0">
          <span>Empty project</span>
          <a href="javascript:;" @click.stop="createFile">Create File</a>
        </div>
      </div>
    </div>

    <!-- Binder tree view TESTING-->
    <div
      class="project-tree" v-if="projectTree && isBinder"
    >
      <div class="title">
        <svg class="icon icon-arrow" :class="{'fold': !showDirectories}" aria-hidden="true" @click.stop="toggleDirectories()">
          <use xlink:href="#icon-arrow"></use>
        </svg>
        <span class="default-cursor text-overflow" @click.stop="toggleDirectories()">{{ projectTree.name }} Test</span>
      </div>
      <div class="tree-wrapper" v-show="showDirectories">
        <draggable v-model="list" @end="onDrop">
          <div v-for="item in binderTree">
          <div v-if="item.isDirectory">
          <folder-b
          :folder="item"
          :depth="getDepth(item.name)"
        ></folder-b>
        <input
          type="text" class="new-input" v-show="createCache.dirname === projectTree.pathname"
          :style="{'margin-left': `${depth * 5 + 15}px` }"
          ref="input"
          v-model="createName"
          @keydown.enter="handleInputEnter"
        >
          </div>
          <div v-else-if="item.isFile">
          <file
          :file="item"
          :depth="getDepth(item.name)"
        ></file>
          </div>
          </div>
      </draggable>
        <div class="empty-project" v-if="projectTree.files.length === 0 && projectTree.folders.length === 0">
          <span>Empty project</span>
          <a href="javascript:;" @click.stop="createFile">Create File</a>
        </div>
      </div>
    </div>
    <div v-else class="open-project">
      <div class="centered-group">
        <svg aria-hidden="true" :viewBox="FolderIcon.viewBox">
          <use :xlink:href="FolderIcon.url"></use>
        </svg>
        <button class="button-primary" @click="openFolder">
          Open Folder
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import Folder from './treeFolder.vue'
import FolderB from './treeFolderB.vue'
import File from './treeFile.vue'
import FileB from './treeFileB.vue'
import OpenedFile from './treeOpenedTab.vue'
import { mapState } from 'vuex'
import bus from '../../bus'
import { createFileOrDirectoryMixins } from '../../mixins'
import FolderIcon from '@/assets/icons/undraw_folder.svg'
import FileIcon from './icon.vue'
import draggable from '../../../../node_modules/vuedraggable'
import fs from 'fs'
import path from 'path'

let nodeConsole = require('console')
let myConsole = new nodeConsole.Console(process.stdout, process.stderr)

export default {
  mixins: [createFileOrDirectoryMixins],
  data () {
    this.depth = 0
    this.FolderIcon = FolderIcon
    return {
      showDirectories: true,
      showNewInput: false,
      showOpenedFiles: true,
      createName: '',
      isBinder: false,
      binderTree: []
    }
  },
  props: {
    projectTree: {
      validator: function (value) {
        return typeof value === 'object'
      },
      required: true,
      default: () => []
    },
    openedFiles: Array,
    tabs: Array
  },
  components: {
    Folder,
    FolderB,
    File,
    FileB,
    FileIcon,
    OpenedFile,
    draggable
  },
  computed: {
    ...mapState({
      createCache: state => state.project.createCache
    })
  },
  created () {
    this.$nextTick(() => {
      bus.$on('SIDEBAR::show-new-input', this.handleInputFocus)
      // hide rename or create input if needed
      document.addEventListener('click', event => {
        const target = event.target
        if (target.tagName !== 'INPUT') {
          this.$store.dispatch('CHANGE_ACTIVE_ITEM', {})
          this.$store.commit('CREATE_PATH', {})
          this.$store.commit('SET_RENAME_CACHE', null)
        }
      })
      document.addEventListener('contextmenu', event => {
        const target = event.target
        if (target.tagName !== 'INPUT') {
          this.$store.commit('CREATE_PATH', {})
          this.$store.commit('SET_RENAME_CACHE', null)
        }
      })
      document.addEventListener('keydown', event => {
        if (event.key === 'Escape') {
          this.$store.commit('CREATE_PATH', {})
          this.$store.commit('SET_RENAME_CACHE', null)
        }
      })
    })
    this.checkBinder()
    this.makeBinderTree()
  },
  watch: {
    projectTree () {
      this.checkBinder()
      this.makeBinderTree()
    },
    isBinder () {
      this.checkBinder()
    }
  },
  methods: {
    openFolder () {
      this.$store.dispatch('ASK_FOR_OPEN_PROJECT')
    },
    saveAll (isClose) {
      this.$store.dispatch('ASK_FOR_SAVE_ALL', isClose)
    },
    createFile () {
      this.$store.dispatch('CHANGE_ACTIVE_ITEM', this.projectTree)
      bus.$emit('SIDEBAR::new', 'file')
    },
    toggleOpenedFiles () {
      this.showOpenedFiles = !this.showOpenedFiles
    },
    toggleDirectories () {
      this.showDirectories = !this.showDirectories
    },
    onDrop (event) {
      myConsole.log('onDrop Test')
    },
    flattenArray (array) {
      let result = []
      array.forEach((item) => {
        result.push(item)
        if (item.isDirectory) {
          const allSubItems = [...item.files, ...item.folders].sort((a, b) => a.name.localeCompare(b.name))
          const children = this.flattenArray(allSubItems)
          children.forEach((child) => {
            result.push(child)
          })
        }
      })
      return result
    },
    renameFiles (arr) {
      let index = 0
      const traverse = (node, parentIndex = '') => {
        if (node.isDirectory) {
          const sortedChildren = [...node.files, ...node.folders].sort((a, b) => a.name.localeCompare(b.name))
          sortedChildren.forEach((child, i) => {
            const childIndex = parentIndex ? parentIndex + '.' + i : index + '.' + i
            traverse(child, childIndex)
          })
        }
        if (node.name) {
          const newName = parentIndex ? parentIndex + '#' + node.name : index + '#' + node.name
          const newPath = node.pathname.substring(0, node.pathname.lastIndexOf('/')) + '/' + newName
          fs.rename(node.pathname, newPath, (err) => {
            if (err) throw err
          })
        }
      }
      arr.forEach((node) => {
        traverse(node)
        index++
      })
      return arr
    },
    makeBinder () {
      if (this.isBinder) {
        myConsole.log('this is a Binder already')
        return this.isBinder
      } else {
        let binderTree = [...this.projectTree.files, ...this.projectTree.folders].sort((a, b) => a.name.localeCompare(b.name))
        this.renameFiles(binderTree)
        let filePath = path.join(this.projectTree.pathname, '.markTextBinder.md')
        fs.appendFile(filePath, 'This is a MarkText Binder!', function (err) {
          if (err) throw err
          this.isBinder = true
          myConsole.log('I did the thing! isBinder =', this.isBinder)
        })
      }
    },
    checkBinder () {
      let folderPath = this.projectTree.pathname
      let fileName = '.markTextBinder.md'
      let filePath = path.join(folderPath, fileName)
      fs.readFile(filePath, 'utf8', (err, data) => {
        if (err) {
          this.isBinder = false
        } else {
          this.isBinder = true
        }
      })
    },
    makeBinderTree () {
      this.binderTree = this.flattenArray(this.projectTree.allItems)
    },
    getDepth (str) {
      myConsole.log('getDepth was triggered.')
      const integerSubstring = str.split('#')[0]
      if (!integerSubstring.includes('.')) {
        return 0
      }
      return integerSubstring.split('.').length - 1
    }
  }
}
</script>

<style scoped>
  .list-item {
    display: inline-block;
    margin-right: 10px;
  }

  .list-enter-active, .list-leave-active {
    transition: all .2s;
  }
  .list-enter, .list-leave-to
  /* .list-leave-active for below version 2.1.8 */ {
    opacity: 0;
    transform: translateX(-50px);
  }
  .tree-view {
    font-size: 14px;
    color: var(--sideBarColor);
    display: flex;
    flex-direction: column;
    height: 100%;
  }
  .tree-view > .title {
    height: 35px;
    line-height: 35px;
    padding: 0 15px;
    display: flex;
    flex-shrink: 0;
    flex-direction: row-reverse;
  }

  .icon-arrow {
    margin-right: 5px;
    transition: all .25s ease-out;
    transform: rotate(90deg);
    fill: var(--sideBarTextColor);
  }

  .icon-arrow.fold {
    transform: rotate(0);
  }

  .opened-files,
  .project-tree {
    & > .title {
      height: 30px;
      line-height: 30px;
      font-size: 14px;
    }
  }

  .opened-files .title {
    padding-right: 15px;
    display: flex;
    align-items: center;
    & > span {
      flex: 1;
    }
    & > a {
      display: none;
      text-decoration: none;
      color: var(--sideBarColor);
      margin-left: 8px;
    }
  }
  .opened-files div.title:hover > a,
  .opened-files div.title > a:hover {
    display: block;
    &:hover {
      color: var(--highlightThemeColor);
    }
  }
  .opened-files {
    display: flex;
    flex-direction: column;
  }
  .default-cursor {
    cursor: pointer;
  }
  .opened-files .opened-files-list {
    max-height: 200px;
    overflow: auto;
    &::-webkit-scrollbar:vertical {
      width: 8px;
    }
    flex: 1;
  }

  .project-tree {
    display: flex;
    flex-direction: column;
    overflow: auto;
    & > .title {
      padding-right: 15px;
      display: flex;
      align-items: center;
      & > span {
        flex: 1;
        user-select: none;
      }
      & > a {
        pointer-events: auto;
        cursor: pointer;
        margin-left: 8px;
        color: var(--sideBarIconColor);
        opacity: 0;
      }
      & > a:hover {
        color: var(--highlightThemeColor);
      }
      & > a.active {
        color: var(--highlightThemeColor);
      }
    }
    & > .tree-wrapper {
      overflow: auto;
      flex: 1;
      &::-webkit-scrollbar:vertical {
        width: 8px;
      }
    }
    flex: 1;
  }
  .project-tree div.title:hover > a {
    opacity: 1;
  }
  .open-project {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: space-around;
    align-items: center;
    padding-bottom: 100px;
    & .centered-group {
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    & svg {
      width: 120px;
      fill: var(--themeColor);
    }
    & button.button-primary {
      display: block;
      margin-top: 20px;
    }
  }
  .new-input {
    outline: none;
    height: 22px;
    margin: 5px 0;
    padding: 0 6px;
    color: var(--sideBarColor);
    border: 1px solid var(--floatBorderColor);
    background: var(--floatBorderColor);
    width: calc(100% - 45px);
    border-radius: 3px;
  }
  .tree-wrapper {
    position: relative;
  }
  .empty-project {
    position: absolute;
    top: 0;
    left: 0;
    font-size: 14px;
    display: flex;
    flex-direction: column;
    padding-top: 40px;
    align-items: center;
    & > a {
      color: var(--highlightThemeColor);
      text-align: center;
      margin-top: 15px;
      text-decoration: none;
    }
  }
  .bold {
    font-weight: 600;
  }
</style>
