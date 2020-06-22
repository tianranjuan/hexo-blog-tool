<template>
  <div class="workspace">
    <a-row :gutter="20" style="height:100%;">
      <a-col :span="6" style="height: 100%;">
        <div>
          <a-button type="link" @click="openfile">
            <a-icon type="folder-open" />
          </a-button>
          <a-divider type="vertical"></a-divider>
          <a-button type="link" @click="newFile">
            <a-icon type="file-add" />
          </a-button>
          <a-divider type="vertical"></a-divider>
          <a-button type="link" @click="publish">
            <a-icon type="cloud-upload"/>
          </a-button>
          <a-divider type="vertical"></a-divider>
          <a-button type="link" @click="reloadFileList">
            <a-icon type="redo" />
          </a-button>
        </div>
        <br/>
        <h3>文件系统</h3>
        <div style="height:calc(100% - 100px);display:flex;overflow:auto;">
          <a-list itemLayout="horizontal" :dataSource="fileList" style="width:100%;">
            <a-list-item slot="renderItem" slot-scope="item,index" :class="{listItem: true, hightlight: fileName === item.name}">
              <a slot="actions" @click.stop="editFile(item, index)">编辑</a>
              <a slot="actions" @click.stop="removeFile(item, index)">删除</a>
              <div>
                {{item.name}}
              </div>
            </a-list-item>
          </a-list>
        </div>
      </a-col>
      <a-col :span="18" style="height:100%;">
        <input class="titleInput" v-model="fileName">
        <mavon-editor :editable="editable" style="height: calc(100% - 50px);z-index:1;" v-model="value" @save="saveFile"/>
      </a-col>
    </a-row>
    <a-modal
      title="发布blog"
      v-model="visible"
      :closable="false"
      :keyboard="false"
      :maskClosable="false"
    >
       <p v-for="str in complieOutput" :key="str">
         {{str}}
       </p>
        <template slot="footer">
          <a-button key="submit" :disabled="pushDone" type="primary"  @click="closePushWindow">
            关闭
          </a-button>
        </template>
    </a-modal>
  </div>
</template>

<script>
import { ipcRenderer } from "electron";
import moment from 'moment';
const fs = require("fs");
const path = require("path");
const random = require("string-random");
const exec = require('child_process').exec
export default {
  name: "app",
  data() {
    return {
      value: "",
      postPath: "",
      themePath: "",
      data: [],
      fileList: [],
      currIndex: -1,
      openingFilePath: '',
      fileName:'',
      editable: false,
      visible: false,
      pushDone: true,
      complieOutput: ['执行命令：hexo clean & hexo g & hexo deploy'],
      outputList: [
        '正在删除 database </br>',
        '正在删除 public 文件夹 </br>',
        '开始转换...',
        '转换中...',
      ],
    };
  },
  async created() {
  },
  methods: {
    async openfile() {
      ipcRenderer.send("selectDirectory");
      ipcRenderer.on("openDirectory", this.getPath);
    },
    getPath: function(e, p) {
      if (p == null) {
        this.$message.info("请选择一个文件夹");
      } else {
        const [path] = p;
        this.path = path
        this.postPath = `${path}\\source\\_posts`;
        this.themePath = `${path}\\themes`;
        this.fileList = this.readFileList(this.postPath, []);
        ipcRenderer.send("changeTitle", path);
      }
    },
    readFileList(dir, filesList = []) {
      const files = fs.readdirSync(dir);
      files.forEach((item, index) => {
        var fullPath = path.join(dir, item);
        const stat = fs.statSync(fullPath);
        if (stat.isDirectory()) {
          readFileList(path.join(dir, item), filesList); //递归读取文件
        } else {
          filesList.push({path: fullPath, name: item.split('.')[0]});
        }
      });
      return filesList;
    },
    editFile(item, index) {
      this.currIndex = index
      this.openingFile = item
      this.openingFilePath = item.path
      this.fileName = item.name
      this.readFile(item.path)
    },
    readFile(path) {
      fs.readFile(path, 'utf-8', (err, data) => {
        this.value = data
        this.editable = true
      })
    },
    newFile() {
      if (!this.path) return alert('请选择一个文件夹')
      const fileName = moment().format('YYYY-MM-DD')
      const template = this.createTemplate(fileName)
      const p = `${this.postPath}\\${fileName}.md`
      fs.writeFile(p, template, (err, data) => {
          if (!err) {
            const file = {
              path: p,
              name: fileName,
            }
            this.fileList.shift(file)
            this.currIndex = 0
            this.value = template
            this.fileName = fileName
            this.openingFile = file
            this.getPath(null, [this.path])
          } else {
            alert('创建失败')
          }
      })
    },
    saveFile(value) {
      if(this.fileName === this.openingFile.name) {
        fs.writeFile(this.openingFile.path, value, (err, data) => {
          console.log('save done');
          this.getPath(null, [this.path])
        })
      } else {
        const p = `${this.postPath}\\${this.fileName}.md`
        fs.rename(this.openingFile.path, p, (err) => {
          if (err) {
            alert('保存异常')
          } else {
            fs.writeFile(p, value, (err, data) => {
              this.openingFile.name = this.fileName
              this.openingFile.path = p
              this.getPath(null, [this.path])
            })
          }
        })
      }
    },
    removeFile(item, index) {
      this.$confirm({
          title: '警告',
          content: `是否确定要删除\`${item.name}.md\`？`,
          okText: '删除',
          cancelText: '取消',
          onOk: () => {
            fs.unlink(item.path, () => {
              if (this.fileName === item.name) {
                this.value = ''
                this.openingFile = {}
                this.fileName = ''
                this.currIndex = -1
                this.editable = false
              }
              this.getPath(null, [this.path])
            })
          }
        });
    },
    createTemplate(fileName) {
      return  `
---
fileName: ${random(12)}
title: ${fileName}
date: ${moment().format('YYYY-MM-DD HH:mm:ss')}
tags:
categories:
keywords: 
---

      `
    },
    publish() {
      if (!this.path) return alert('请选择一个文件夹')
      this.visible = true
      const workerProcess = exec('hexo clean & hexo g & hexo deploy', {cwd: this.path})
      let i = 0
      workerProcess.stdout.on('data', (data) => {
        // this.complieOutput += `${data}</br>`
        if (i < 4) {
          this.complieOutput.push(`${this.outputList[i]}`)
          i++
        }
      });

      // 打印错误的后台可执行程序输出
      workerProcess.stderr.on('data', (data) => {
        // this.complieOutput += `${data}</br>`
      });
      
      // 退出之后的输出
      workerProcess.on('close', (code) => {
        this.complieOutput.push(`推送完成`)
        this.pushDone = false
      })

      // shell.exec(`hexo clean & hexo g & hexo deploy`);
      // const cmd = `cd ${this.path} && npm run push`
      // exec(cmd, function(error, stdout, stderr) {
      //   if(error){
      //       console.log(error);
      //   }
      //   else{
      //       console.log("成功");
      //   }
    // });
    },
    closePushWindow() {
      this.complieOutput = ''
      this.visible = false
      this.pushDone = true
    },
    reloadFileList() {
      if (this.path) {
        this.getPath(null, [this.path])
      }
    },
  }
};
</script>

<style>
.workspace {
  height: 100%;
  padding: 12px;
}

h3 {
  background: rgba(0, 0, 0, 0.05);
  padding: 5px;
}

.listItem:hover {
  background: rgba(0, 0, 0, 0.025);
}

.hightlight{
  background: rgba(0, 0, 0, 0.025);
}

.titleInput{
  height:48px;
  line-height:48px;
  font-size: 30px;
  border: none;
  width: 100%;
  margin-bottom: 2px;
}

</style>