<template>
  <el-dialog
    custom-class="tab-title-dialog add-task-dialog"
    width="64vw"
    :visible.sync="visible"
    :before-close="handleClose"
    @open="handleOpen"
    @closed="handleClosed"
  >
    <el-form ref="taskForm" label-position="left" :model="form" :rules="rules">
      <el-tabs value="makeTorrent">
        <el-tab-pane  :label="$t('task.make-torrent') || '制作种子'" name="makeTorrent">
          <el-form-item>
            <div class="selective-torrent">
              <el-row>
                <el-upload
                  class="torrent-upload"
                  ref="upload"
                  action="/"
                  :on-change="uploadChange"
                  :on-exceed="uploadExceed"
                  :auto-upload="false"
                  :show-file-list="false">
                  <el-button size="mini" type="primary">{{ $t('task.make-torrent-add-file') || '添加文件' }}</el-button>
                </el-upload>
                <el-button size="mini" type="primary" @click="removeFiles">{{ $t('task.make-torrent-remove-file') || '移除' }}</el-button>
              </el-row>
              <mo-task-files
                ref="torrentFileList"
                mode="ADD"
                :files="files"
                :height="200"
                @selection-change="handleSelectionChange"
              />
            </div>
          </el-form-item>
        </el-tab-pane>
      </el-tabs>
      <el-form-item
        :label="`${$t('task.task-dir')}: `"
        :label-width="formLabelWidth"
      >
        <el-input
          placeholder=""
          v-model="form.dir"
          :readonly="isMas"
        >
          <mo-select-directory
            v-if="isRenderer"
            slot="append"
            @selected="onDirectorySelected"
          />
        </el-input>
      </el-form-item>
    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-row>
        <el-col :span="24" :xs="24">
          <el-button @click="handleCancel('taskForm')">
            {{$t('app.cancel')}}
          </el-button>
          <el-button
            type="primary"
            @click="submitForm('taskForm')"
          >
            {{ $t('task.make-torrent-save') || '保存种子' }}
          </el-button>
        </el-col>
      </el-row>
    </div>
  </el-dialog>
</template>

<script>
  import is from 'electron-is'
  import createTorrent from 'create-torrent'
  import fs from 'fs'
  import { mapState } from 'vuex'
  import SelectDirectory from '@/components/Native/SelectDirectory'
  import {
    initTaskForm
  } from '@/utils/task'
  import {
    buildFileList,
    getFileExtension
  } from '@shared/utils'
  import '@/components/Icons/inbox'
  import TaskFiles from '@/components/TaskDetail/TaskFiles'

  export default {
    name: 'mo-make-torrent',
    components: {
      [SelectDirectory.name]: SelectDirectory,
      [TaskFiles.name]: TaskFiles
    },
    props: {
      visible: {
        type: Boolean,
        default: false
      }
    },
    data () {
      return {
        files: [],
        opts: {},
        formLabelWidth: '100px',
        form: {},
        rules: {}
      }
    },
    computed: {
      isRenderer: () => is.renderer(),
      isMas: () => is.mas(),
      ...mapState('app', {
        makeTorrentFiles: state => state.makeTorrentFiles,
        taskList: state => state.taskList
      }),
      ...mapState('preference', {
        config: state => state.config
      })
    },
    watch: {
      makeTorrentFiles (fileList) {
        if (fileList.length === 0) {
          this.fileReset()
          return
        }

        for (const i in fileList) {
          if (!fileList[i].raw) {
            return
          }
        }

        this.files = fileList.map((file, index) => {
          return {
            uid: file.uid,
            idx: index + 1,
            extension: getFileExtension(file.raw.path),
            length: file.size,
            name: file.name,
            path: file.raw.path,
            offset: index,
            raw: file.raw
          }
        })
        this.$refs.torrentFileList.toggleAllSelection()
      },
      visible (current) {
        if (current === true) {
          document.addEventListener('keydown', this.handleHotkey)
        } else {
          document.removeEventListener('keydown', this.handleHotkey)
        }
      }
    },
    methods: {
      fileReset () {
        this.files = []
        if (this.$refs.torrentFileList) {
          this.$refs.torrentFileList.clearSelection()
        }
        this.$refs.upload.uploadFiles = []
      },
      uploadChange (file, fileList) {
        if (this.form.dir.indexOf('.torrent') === -1) {
          this.form.dir += (this.form.dir.substring(this.form.dir.length - 1) !== '/' ? '/' : '') + file.name + '.torrent'
        }
        this.$store.dispatch('app/makeTorrentAddFile', { fileList })
      },
      uploadExceed (files) {
        const fileList = buildFileList(files[0])
        this.$store.dispatch('app/makeTorrentAddFile', { fileList })
      },
      handleSelectionChange (val) {
      },
      handleOpen () {
        this.form = initTaskForm(this.$store.state)
      },
      handleCancel (formName) {
        this.$store.dispatch('app/hideMakeTorrentDialog')
      },
      handleClose (done) {
        this.$store.dispatch('app/hideMakeTorrentDialog')
      },
      handleClosed () {
        this.reset()
      },
      handleHotkey (event) {
        if (event.key === 'Enter' && (event.ctrlKey || event.metaKey)) {
          event.preventDefault()

          this.submitForm('taskForm')
        }
      },
      onDirectorySelected (dir) {
        this.form.dir = dir
      },
      removeFiles () {
        const toRemoveIdx = this.$refs.torrentFileList.selectedFiles.map(file => file.idx)
        const toRemoveUid = this.$refs.torrentFileList.selectedFiles.map(file => file.uid)

        this.files = this.files.filter(file => toRemoveIdx.indexOf(file.idx) === -1)
        this.$refs.torrentFileList.toggleAllSelection()

        const fileList = this.makeTorrentFiles.filter(file => toRemoveUid.indexOf(file.uid) === -1)
        this.$store.dispatch('app/makeTorrentAddFile', { fileList })
        this.$refs.upload.uploadFiles = fileList
      },
      reset () {
        this.fileReset()
        this.form = initTaskForm(this.$store.state)
      },
      submitForm (formName) {
        const fileList = this.files.map(file => file.raw)
        const opts = this.opts
        const dir = this.form.dir
        this.$refs[formName].validate(valid => {
          if (!valid) {
            return false
          }

          createTorrent(fileList, opts, (err, torrent) => {
            if (!err) {
              fs.writeFile(dir, torrent, (fsErr) => {
                if (!fsErr) {
                  this.$store.dispatch('app/hideMakeTorrentDialog')
                } else {
                  console.log(fsErr)
                }
              })
            }
          })
        })
      }
    }
  }
</script>

<style lang="scss">
.el-dialog.add-task-dialog {
  max-width: 632px;
  min-width: 380px;
  .task-advanced-options .el-form-item:last-of-type {
    margin-bottom: 0;
  }
  .el-tabs__header {
    user-select: none;
  }
  .el-input-number.el-input-number--mini {
    width: 100%;
  }
  .help-link {
    font-size: 12px;
    line-height: 14px;
    padding-top: 7px;
    > a {
      color: #909399;
    }
  }
  .el-dialog__footer {
    padding-top: 20px;
    background-color: $--add-task-dialog-footer-background;
    border-radius: 0 0 5px 5px;
  }
  .dialog-footer {
    .chk {
      float: left;
      line-height: 28px;
      &.el-checkbox {
        & .el-checkbox__input {
          line-height: 19px;
        }
        & .el-checkbox__label {
          padding-left: 6px;
        }
      }
    }
  }
}
.selective-torrent {
  .el-row {
    margin-bottom: 10px;
  }
  .torrent-upload {
    display: inline;
  }
  .torrent-name {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
  }
  .torrent-info {
    margin-bottom: 15px;
    font-size: 12px;
    line-height: 16px;
  }
  .torrent-actions {
    text-align: right;
    line-height: 16px;
    &> span {
      cursor: pointer;
      display: inline-block;
      vertical-align: middle;
      height: 14px;
      padding: 1px;
    }
  }
}
</style>
