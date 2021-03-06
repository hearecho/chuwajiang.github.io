<template>
  <article v-loading.full="loading">
    <el-form label-width="50px">
      <el-row :gutter="12" v-if="isMarkdown()">
        <el-col :md="12">
          <el-input placeholder="请输入文章标题" v-model="title" :size="size"></el-input>
        </el-col>
        <el-col :xs="24" :sm="14" :md="6">
          <el-input placeholder="存放文件夹, 可以为空" v-model="path" :size="size">
             <template slot="prepend">{{ prefix }}</template>
          </el-input>
        </el-col>
        <el-col :xs="24" :sm="10" :md="6">
          <el-button type="primary" @click="save" :size="size">保存</el-button>
          <el-button @click="newPost" icon="plus" :size="size"></el-button>
          <el-button type="danger" icon="delete" @click="confirmRemove" :size="size"></el-button>
        </el-col>
      </el-row>
      <el-row :gutter="12" v-else>
        <el-col :span="20">
          <el-input readonly v-model="downloadUrl" ref="copyText" :size="size">
            <template slot="append">
              <el-button @click="copy" :size="size">复制文件链接</el-button>
            </template>
          </el-input>
        </el-col>
        <el-col :span="4">
          <el-button type="danger" icon="delete" @click="confirmRemove" :size="size"></el-button>
        </el-col>
      </el-row>
    </el-form>
    <mavon-editor v-show="isMarkdown()" v-model="content" class="content" :default_open="defaultOpen" @save="save" @imgAdd="imgAdd" :toolbars="toolbars" ref="editor" />
    <img v-show="!isMarkdown()" class="attachment" :src="downloadUrl" />
  </article>
</template>
<script>
import { Base64 } from 'js-base64';
import { mavonEditor } from 'mavon-editor';
import 'mavon-editor/dist/css/index.css';
import { repo } from '../../api';
import EventBus from '../../event-bus';
import ImageCompressor from 'image-compressor';
const smallDevice = window.innerWidth > 1100;

export default {
  data() {
    return {
      title: this.initTitle(),
      content: '',
      path: '',
      prefix: '我的文档',
      downloadUrl: '',
      sha: '',
      loading: false,
      defaultOpen: smallDevice ? 'preview' : 'edit',
      size: smallDevice ? 'large' : 'small',
      toolbars: smallDevice ? undefined : {
        bold: true,
        italic: true,
        header: true,
        mark: true,
        undo: true,
        redo: true,
        code: true,
        quote: true,
        fullscreen: true,
        imagelink: true,
        preview: true,
        readmodel: true,
        save: true
      }
    };
  },
  created() {
    this.fetchFile();
    window.onpagehide = window.onunload = window.onbeforeunload = () => {
      this.saveDraft();
    };
  },
  components: { mavonEditor },
  methods: {
    saveDraft() {
      try {
        localStorage.setItem(this.title, this.content);
      } catch (e) {}
    },
    getTime(time) {
      const date = new Date(time);
      return `${date.getFullYear()}${this.pad(date.getMonth() + 1)}${this.pad(date.getDate())}${this.pad(date.getHours())}${this.pad(date.getMinutes())}${this.pad(date.getMilliseconds())}`;
    },
    isMarkdown(path = this.title) {
      return /\.md$/.test(path) || path.trim() === '';
    },
    fetchFile() {
      if (!this.$route.query.path) {
        this.initContent();
        return;
      };
      this.loading = true;
      this.downloadUrl = '';
      repo.contents(this.$route.query.path + '?rd=' + Math.random())
      .fetch()
      .then(({ path, content, sha, name, downloadUrl }) => {
        path = path.split('/');
        this.originTitle = this.title = path.pop();
        this.prefix = path.shift() === 'media' ? '我的图片' : '我的文档';
        this.path = path.join('/') || '';
        this.sha = sha;
        this.downloadUrl = downloadUrl;
        this.initContent(content);
        this.loading = false;
      })
      .catch((err = {}) => {
        const message = /"message": "([^"]+)/m.test(err.message) && RegExp.$1 || err.toString();
        this.loading = false;
        if (~message.indexOf('not found')) {
          this.content = '';
          return;
        }
        this.$message.error(message.trim());
      });
    },
    initContent(content) {
      if (!this.content) {
        if (this.isMarkdown(this.title)) {
          if (content) {
            content = Base64.decode(content);
          }
          let draft = '';
          if ((draft = localStorage.getItem(this.title))) {
            if (!content || !draft) {
              this.content = draft || content || '';
            } else if (draft !== content) {
              this.$confirm('检测到本地草稿与线上内容不一致，是否使用本地草稿？')
              .then(() => {
                this.content = draft;
              })
              .catch(() => {
                this.content = content;
              });
            } else {
              this.content = content;
            }
            this.$nextTick(() => {
              document.querySelector('.admin-body').scrollLeft = 1000;
            });
          } else if (content) {
            this.content = content;
          }
        } else {
          this.content = content;
        }
      }
      this.originContent = this.content;
    },
    save() {
      if (!this.title) {
        this.$message.error('请输入文件名');
        return;
      }
      this.title = this.title.replace(/[\/\\]/g, '');
      let path = [
        '_posts',
        this.path.replace(/(^\/|\/$)/g, ''),
        this.title
      ].join('/').replace(/\/\/+/g, '/');

      const config = {
        path,
        message: 'update file: ' + path,
        content: this.isMarkdown() ? Base64.encode(this.content) : this.content
      };
      if (this.sha) {
        config.sha = this.sha;
      } else {
        config.message = 'create file: ' + path;
      }
      this.loading = true;
      return this.upload(config)
      .then(() => {
        this.$message({
          type: 'success',
          message: '保存成功'
        });
        this.saveDraft();
        if (this.originTitle && this.originTitle !== this.title) {
          this.removeFile();
        }
        this.originContent = this.content;
      })
      .catch((err = {}) => {
        const message = /"message": "([^"]+)/m.test(err.message) && RegExp.$1 || err.toString();
        if (~message.indexOf('does not match')) {
          this.fetchFile();
        } else {
          this.loading = false;
          this.$message.error(message.trim());
        }
      });
    },
    upload(config) {
      return repo.contents(config.path).add(config)
      .then((response) => {
        this.loading = false;
        EventBus.$emit('updateFiles');
        return response;
      });
    },
    pad(num) {
      return num > 9 ? num : '0' + num;
    },
    initTitle() {
      const now = new Date();
      return `${now.getFullYear()}-${this.pad(now.getMonth() + 1)}-${this.pad(now.getDate())}-请修改标题.md`;
    },
    reset() {
      this.title = this.initTitle();
      this.originTitle = '';
      this.content = '';
      this.downloadUrl = '';
      this.prefix = '我的文档';
      this.originContent = '';
      this.path = '';
      this.sha = null;
    },
    confirmRemove() {
      return this.$confirm('是否要删除此文件？')
      .then(this.removeFile);
    },
    removeFile() {
      this.loading = true;
      return repo.contents(this.$route.query.path)
      .remove({
        message: 'remove file',
        sha: this.sha
      })
      .then(() => {
        this.loading = false;
        this.$router.replace({
          path: '/'
        });
        EventBus.$emit('updateFiles');
      })
      .catch((err = {}) => {
        this.loading = false;
        this.$message.error(/"message": "([^"]+)/m.test(err.message) && RegExp.$1 || err.toString());
      });
    },
    newPost() {
      if (this.originContent !== this.content) {
        return this.$confirm('是否放弃原有文章？').then(this.reset);
      } else {
        this.$router.replace({
          path: '/'
        });
        this.reset();
      }
    },
    getBase64(file) {
      return new Promise((resolve, reject) => {
        new ImageCompressor(file, { // eslint-disable-line
          quality: 0.6,
          success(result) {
            var reader = new FileReader();
            reader.readAsDataURL(result);
            reader.onload = resolve;
            reader.onerror = reject;
          },
          error(e) {
            reject(e);
          }
        });
      });
    },
    imgAdd(filename, file) {
      this.loading = true;
      const name = 'media/' + this.getTime(Date.now()) + '.' + file.type.split('/').pop();
      this.getBase64(file)
      .then((res) => {
        return this.upload({
          path: name,
          message: 'upload image: ' + file.name,
          content: res.target.result.slice(res.target.result.indexOf(',') + 1)
        });
      })
      .then(({ content }) => {
        this.$refs['editor'].$img2Url(filename, content.downloadUrl);
      })
      .catch((err = {}) => {
        this.loading = false;
        this.$message.error(/"message": "([^"]+)/m.test(err.message) && RegExp.$1 || err.toString());
      });
    },
    copy() {
      var copyTextarea = this.$refs['copyText'].$refs['input'];
      copyTextarea.select();
      try {
        var successful = document.execCommand('copy');
        var msg = successful ? '成功' : '失败';
        this.$message('复制' + msg);
      } catch (err) {
        this.$message.error('请手工复制链接');
      }
    }
  },
  watch: {
    '$route.query'(val) {
      this.reset();
      this.fetchFile();
    }
  }
};
</script>
<style lang="scss" scoped>
.el-col {
  margin-bottom: 8px;
}
.attachment {
  object-fit: contain;
  max-height: 80%;
}
article {
  flex: 1;
  display: flex;
  flex-direction: column;
}
.content {
  z-index: 10;
  flex: 1;
  min-width: 100%;
  max-width: 100%;
}
</style>
<style lang="scss">
.mu-item-wrapper .mu-item {
  min-height: auto;
}
.el-message {
  max-width: 100%;
  .el-message__group {
    height: auto;
    p {
      white-space: pre-wrap;
      word-break: break-all;
    }
  }
}
</style>