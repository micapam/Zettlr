<template>
  <div id="sidebar">
    <TabBar
      v-bind:tabs="tabs"
      v-bind:current-tab="currentTab"
      v-on:tab="setCurrentTab($event)"
    ></TabBar>

    <!-- Now the tab containers -->
    <div
      v-if="currentTab === 'toc'"
      role="tabpanel"
    >
      <!-- Table of Contents -->
      <h1>{{ titleOrTocLabel }}</h1>
      <!-- Show the ToC entries -->
      <div
        v-for="(entry, idx) of tableOfContents"
        v-bind:key="idx"
        class="toc-entry-container"
        v-bind:style="{
          'margin-left': `${entry.level * 10}px`
        }"
        v-on:click="($root as any).jtl(entry.line, true)"
      >
        <div class="toc-level">
          {{ entry.renderedLevel }}
        </div>
        <div
          v-bind:class="{ 'toc-entry': true, 'toc-entry-active': tocEntryIsActive(entry.line, idx) }"
          v-bind:data-line="entry.line"
          v-html="toc2html(entry.text)"
        ></div>
      </div>
    </div>

    <div
      v-if="currentTab === 'references'"
      role="tabpanel"
    >
      <!-- References -->
      <h1>{{ referencesLabel }}</h1>
      <!-- Will contain the actual HTML -->
      <div v-html="referenceHTML"></div>
    </div>

    <div
      v-if="currentTab === 'relatedFiles'"
      role="tabpanel"
    >
      <h1>{{ relatedFilesLabel }}</h1>
      <div class="related-files-container">
        <div v-if="relatedFiles.length === 0">
          {{ noRelatedFilesMessage }}
        </div>
        <div v-else>
          <RecycleScroller
            v-slot="{ item, index }"
            v-bind:items="scrollerRelatedFiles"
            v-bind:item-size="43"
            v-bind:emit-update="true"
            v-bind:page-mode="true"
          >
            <div
              v-bind:key="index"
              v-bind:class="{
                'related-file': true,
                'tags': item.props.tags.length > 0,
                'inbound': item.props.link === 'inbound',
                'outbound': item.props.link === 'outbound',
                'bidirectional': item.props.link === 'bidirectional'
              }"
              v-on:mousedown.stop="requestFile($event, item.props.path)"
              v-on:dragstart="beginDragRelatedFile($event, item.props.path)"
            >
              <span
                class="filename"
                draggable="true"
              >{{ getRelatedFileName(item.props.path) }}</span>
              <span class="icons">
                <clr-icon
                  v-if="item.props.tags.length > 0"
                  shape="tag"
                  title="This relation is based on tag similarity."
                ></clr-icon>
                <clr-icon
                  v-if="item.props.link === 'inbound'"
                  shape="arrow left"
                  title="This relation is based on a backlink."
                ></clr-icon>
                <clr-icon
                  v-else-if="item.props.link === 'outbound'"
                  shape="arrow right"
                  title="This relation is based on an outbound link."
                ></clr-icon>
                <clr-icon
                  v-else-if="item.props.link === 'bidirectional'"
                  shape="two-way-arrows"
                  title="This relation is based on a bidirectional link."
                ></clr-icon>
              </span>
            </div>
          </RecycleScroller>
        </div>
      </div>
    </div>

    <div
      v-if="currentTab === 'attachments'"
      role="tabpanel"
    >
      <!-- Other files contents -->
      <h1>
        {{ otherFilesLabel }}
        <clr-icon
          id="open-dir-external"
          v-bind:title="openDirLabel"
          shape="folder"
          class="is-solid"
        ></clr-icon>
      </h1>

      <!-- Render all attachments -->
      <p v-if="attachments.length === 0">
        {{ noAttachmentsMessage }}
      </p>
      <template v-else>
        <a
          v-for="(attachment, idx) in attachments"
          v-bind:key="idx"
          class="attachment"
          draggable="true"
          v-bind:data-link="attachment.path"
          v-bind:data-hash="attachment.hash"
          v-bind:title="attachment.path"
          v-bind:href="`safe-file://${attachment.path}`"
          v-on:dragstart="handleDragStart($event, attachment.path)"
        >
          <span v-html="getIcon(attachment.path)"></span>
          {{ attachment.name }}
        </a>
      </template>
    </div>
  </div>
</template>

<script lang="ts">
/**
 * @ignore
 * BEGIN HEADER
 *
 * Contains:        Sidebar
 * CVM-Role:        View
 * Maintainer:      Hendrik Erz
 * License:         GNU GPL v3
 *
 * Description:     This component renders the sidebar.
 *
 * END HEADER
 */

import { trans } from '@common/i18n-renderer'
import { ClarityIcons } from '@clr/icons'
import TabBar from '@common/vue/TabBar.vue'
import { defineComponent } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'
import { DirMeta, MDFileMeta, OtherFileMeta } from '@dts/common/fsal'
import { TabbarControl } from '@dts/renderer/window'
import sanitizeHtml from 'sanitize-html'
import { getConverter } from '@common/util/md-to-html'
import { RelatedFile } from '@dts/renderer/misc'

const path = window.path
const ipcRenderer = window.ipc

// Must be instantiated after loading, i.e. when the Sidebar is initialized
let md2html: Function

export default defineComponent({
  name: 'MainSidebar',
  components: {
    TabBar,
    RecycleScroller
  },
  data: function () {
    return {}
  },
  computed: {
    /**
     * The Vue Virtual Scroller component expects an array of objects which
     * expose two properties: id and "props". The latter contains the actual
     * object (i.e. the RelatedFile). We may want to merge this functionality
     * into the RelatedFiles generation later on, but this is the safest way
     * for now.
     *
     * @return  {{ id: number, props: RelatedFile }}  The data for the scroller
     */
    scrollerRelatedFiles: function (): any {
      return this.relatedFiles.map((elem, idx) => {
        return { id: idx, props: elem }
      })
    },
    currentTab: function (): string {
      return this.$store.state.config['window.currentSidebarTab']
    },
    tabs: function (): TabbarControl[] {
      return [
        {
          icon: 'indented-view-list',
          id: 'toc',
          target: 'sidebar-toc',
          label: this.tocLabel
        },
        {
          icon: 'book',
          id: 'references',
          target: 'sidebar-bibliography',
          label: this.referencesLabel
        },
        {
          icon: 'file-group',
          id: 'relatedFiles',
          target: 'sidebar-related-files',
          label: this.relatedFilesLabel
        },
        {
          icon: 'attachment',
          id: 'attachments',
          target: 'sidebar-files',
          label: this.otherFilesLabel
        }
      ]
    },
    otherFilesLabel: function (): string {
      return trans('gui.other_files')
    },
    referencesLabel: function (): string {
      return trans('gui.citeproc.references_heading')
    },
    tocLabel: function (): string {
      return trans('gui.table_of_contents')
    },
    relatedFilesLabel: function (): string {
      return trans('gui.related_files_label')
    },
    openDirLabel: function (): string {
      return trans('gui.attachments_open_dir')
    },
    noAttachmentsMessage: function (): string {
      return trans('gui.no_other_files')
    },
    noRelatedFilesMessage: function (): string {
      return trans('gui.no_related_files')
    },
    attachments: function (): OtherFileMeta[] {
      const currentDir = this.$store.state.selectedDirectory as DirMeta|null
      if (currentDir === null) {
        return []
      } else {
        const extensions: string[] = this.$store.state.config.attachmentExtensions
        const attachments = currentDir.children.filter(child => child.type === 'other') as OtherFileMeta[]
        return attachments.filter(attachment => extensions.includes(attachment.ext))
      }
    },
    activeFile: function (): MDFileMeta|null {
      return this.$store.state.activeFile
    },
    relatedFiles: function (): RelatedFile[] {
      return this.$store.state.relatedFiles
    },
    /**
     * Returns either the title property for the active file or the generic ToC
     * label -- to be used within the ToC of the sidebar
     *
     * @return  {string}  The title for the ToC sidebar
     */
    titleOrTocLabel: function (): string {
      if (this.activeFile === null || this.activeFile.frontmatter == null) {
        return this.tocLabel
      }

      const frontmatter = this.activeFile.frontmatter

      if ('title' in frontmatter && frontmatter.title.length > 0) {
        return this.activeFile.frontmatter.title
      } else {
        return this.tocLabel
      }
    },
    modifiedFiles: function (): string[] {
      return this.$store.state.modifiedDocuments
    },
    tableOfContents: function (): any|null {
      return this.$store.state.tableOfContents
    },
    citationKeys: function (): string[] {
      return this.$store.state.citationKeys
    },
    bibliography: function (): any {
      return this.$store.state.bibliography
    },
    referenceHTML: function (): string {
      if (this.bibliography === undefined || this.bibliography[1].length === 0) {
        return `<p>${trans('gui.citeproc.references_none')}</p>`
      } else {
        const html = [this.bibliography[0].bibstart]

        for (const entry of this.bibliography[1]) {
          html.push(entry)
        }

        html.push(this.bibliography[0].bibend)

        return html.join('\n')
      }
    },
    useH1: function (): boolean {
      return this.$store.state.config.fileNameDisplay.includes('heading')
    },
    useTitle: function (): boolean {
      return this.$store.state.config.fileNameDisplay.includes('title')
    },
    displayMdExtensions: function (): boolean {
      return this.$store.state.config['display.markdownFileExtensions']
    }
  },
  watch: {
    modifiedFiles: function () {
      if (this.activeFile == null) {
        return
      }

      // Update the related files when the current document is not modified to
      // immediately account for any changes in the related files.
      const activePath = this.activeFile.path
      if (!(activePath in this.modifiedFiles)) {
        this.$store.dispatch('updateRelatedFiles')
          .catch(e => console.error('Could not update related files', e))
      }
    }
  },
  mounted: function () {
    ipcRenderer.on('links', () => {
      this.$store.dispatch('updateRelatedFiles')
        .catch(e => console.error('Could not update related files', e))
    })
  },
  created: function () {
    // Instantiate a converter so that we can convert the md of our ToC entries
    // to html with citation support
    md2html = getConverter(window.getCitation)
  },
  methods: {
    setCurrentTab: function (which: string) {
      (global as any).config.set('window.currentSidebarTab', which)
    },
    getIcon: function (attachmentPath: string) {
      const fileExtIcon = ClarityIcons.get('file-ext')
      if (typeof fileExtIcon === 'string') {
        return fileExtIcon.replace('EXT', path.extname(attachmentPath).slice(1, 4))
      } else {
        return ''
      }
    },
    /**
     * Adds additional data to the dragevent
     *
     * @param   {DragEvent}  event           The drag event
     * @param   {string}  attachmentPath  The path to add as a file
     */
    handleDragStart: function (event: DragEvent, attachmentPath: string) {
      // Indicate with custom data that this is a file from the sidebar
      event.dataTransfer?.setData('text/x-zettlr-other-file', attachmentPath)
    },
    requestFile: function (event: MouseEvent, filePath: string) {
      ipcRenderer.invoke('application', {
        command: 'open-file',
        payload: {
          path: filePath,
          newTab: event.type === 'mousedown' && event.button === 1
        }
      })
        .catch(e => console.error(e))
    },
    getRelatedFileName: function (filePath: string) {
      const descriptor = this.$store.getters.file(filePath)
      if (descriptor === undefined) {
        return filePath
      }

      if (this.useTitle && descriptor.frontmatter !== null && typeof descriptor.frontmatter.title === 'string') {
        return descriptor.frontmatter.title
      } else if (this.useH1 && descriptor.firstHeading !== null) {
        return descriptor.firstHeading
      } else if (this.displayMdExtensions) {
        return descriptor.name
      } else {
        return descriptor.name.replace(descriptor.ext, '')
      }
    },
    beginDragRelatedFile: function (event: DragEvent, filePath: string) {
      const descriptor = this.$store.getters.file(filePath)

      event.dataTransfer?.setData('text/x-zettlr-file', JSON.stringify({
        type: descriptor.type, // Can be file, code, or directory
        path: descriptor.path,
        id: descriptor.id // Convenience
      }))
    },
    /**
     * Whether the cursor is within the corresponding document section
     *
     * @param   {number}  tocEntryLine          Line number of section heading
     * @param   {number}  tocEntryIdx           Index of heading in ToC
     */
    tocEntryIsActive: function (tocEntryLine: number, tocEntryIdx: number) {
      const cursorLine = this.$store.state.activeDocumentInfo.cursor.line

      // Determine index of next heading in ToC list
      const nextTocEntryIdx = Math.min(tocEntryIdx + 1, this.tableOfContents.length - 1)

      // Now, determine the next heading's line number
      let nextTocEntryLine = Infinity
      if (tocEntryIdx !== nextTocEntryIdx) {
        nextTocEntryLine = this.tableOfContents[nextTocEntryIdx].line
      }

      // True, when cursor lies between current and next heading
      return (cursorLine >= tocEntryLine && cursorLine < nextTocEntryLine)
    },
    /**
     * Converts a Table of Contents-entry to (safe) HTML
     *
     * @param   {string}  entryText  The Markdown ToC entry
     *
     * @return  {string}             The safe HTML string
     */
    toc2html: function (entryText: string): string {
      const html = md2html(entryText)
      return sanitizeHtml(html, {
        // Headings may be emphasised and contain code
        allowedTags: [ 'em', 'kbd', 'code' ]
      })
    }
  }
})
</script>

<style lang="less">
body {
  @button-margin: 5px;
  @border-radius: 5px;
  @button-size: 5px;
  @button-icon-size: 5px;

  #sidebar {
    background-color: rgb(230, 230, 230);
    height: 100%;
    width: 100%;
    overflow-y: auto;
    overflow-x: hidden;

    #open-dir-external {
      padding: @button-margin;
      border-radius: @border-radius;
      display: inline-block;
      width: @button-size;
      height: @button-size;

      clr-icon {
        width: @button-icon-size;
        height: @button-icon-size;
      }
    }

    h1 {
      padding: 10px;
      font-size: 16px;
    }

    p { padding: 10px; }

    a.attachment {
      display: block;
      margin: 10px;
      padding: 4px;
      text-decoration: none;
      color: inherit;
      // Padding 4px + 4px margin + 24px icon width = 32px
      text-indent: -32px;
      padding-left: 32px;
      // Some filenames are too long for the sidebar. However, unlike with the
      // file manager where we have the full filename visible in multiple places,
      // here we must make sure the filename is fully visible. Hence, we don't
      // use white-space: nowrap, but rather word-break: break-all.
      word-break: break-all;

      svg {
        width: 24px;
        height: 24px;
        margin-right: 4px;
        vertical-align: bottom;
        margin-bottom: -1px;
        // Necessary to give the extension icons the correct colour
        fill: currentColor;
      }
    }

    // Bibliography entries
    div.csl-bib-body {
      div.csl-entry {
        display: list-item;
        list-style-type: square;
        margin: 1em 0.2em 1em 1.8em;
        font-size: 80%;
        user-select: text;
        cursor: text;
      }

      a { color: var(--blue-0); }
    }

    // Table of Contents entries
    div.toc-entry-container {
      // Clever calculation based on the data-level property
      // margin-left: calc(attr(data-level) * 10px);
      display: flex;
      margin-bottom: 10px;
      margin-right: 10px;

      div.toc-level {
        flex-shrink: 1;
        padding: 0px 5px;
        font-weight: bold;
        color: var(--system-accent-color, --c-primary);
      }

      div.toc-entry {
        flex-grow: 3;
        cursor: pointer;
        &:hover { text-decoration: underline; }
      }

      div.toc-entry-active {
        font-weight: bold;
        color: var(--system-accent-color);
      }
    }

    div.related-files-container {
      padding: 10px;

      div.related-file {
        // NOTE: The margin + height equal 42, which was the automatic height
        // before we fixed it here. We have to fix it because the Recycle
        // scroller requires completely fixed heights.
        margin-bottom: 10px;
        display: flex;
        align-items: center;
        padding: 5px 5px;
        height: 35px;
        overflow: hidden;

        &:hover { background-color: rgb(200, 200, 200); }

        span.filename {
          font-size: 11px;
          height: 28px;
          flex-grow: 8;
          // The next four properties together ensure that we show at most two
          // lines with a nice ellipsis (…) to indicate cut-off lines.
          overflow: hidden;
          display: -webkit-box;
          -webkit-line-clamp: 2;
          -webkit-box-orient: vertical;
        }

        span.icons {
          display: inline-block;
          border-radius: 4px;
          padding: 2px;
          flex-grow: 2;
          flex-shrink: 0;
          text-align: right;
        }
      }
    }
  }

  &.dark {
    #sidebar {
      background-color: rgba(30, 30, 30, 1);
      color: rgb(230, 230, 230);

      div.related-files-container div.related-file:hover {
        background-color: rgb(80, 80, 80);
      }
    }
  }
}

body.darwin {
  div#sidebar {
    // On macOS the toolbar is 40px high and the documents titlebar is 30px high,
    // so we want to offset the sidebar by that.
    top: calc(40px + 30px);
    background-color: transparent;

    div.related-files-container {
      div.related-file { border-radius: 4px; }
    }
  }

  &.dark {
    div#sidebar {
      background-color: transparent;
    }
  }
}
</style>
