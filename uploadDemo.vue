<template>
  <div class="container">
     <!-- html部分，这里运用onchange事件作为整个事件的开始节点-->
      <input :id="'key' + index" name="key" type="hidden" :value="fileKey">
      <input id="token" name="token" type="hidden" :value="token" class="myToken">
      <input :id="'upload_file' + index" type="file" class="file_btn" @change="fileSubmitAuto" :multiple="multiple" :accept="accept" name="file">
      <input type="button" class="submit_btn" @click="uploadByVueResource">
    <span :class="icon" class="click_area" @click.stop="clickToUpload"></span>
  </div>
</template>

<script>
  /********引用一些工程中的常量以便调用接口，常量根据实际开发情况做相应改变*****/
  import { QINIU } from '../../constants/URL'
  import uuidV1 from '../../../node_modules/uuid/v1.js'

  export default{
    data () {
      return ({
        token: '',
        fileKey: ''
      })
    },
    props: {
      /*
      根据使用环境不同可改变的参数
       */
      /*********************************************************/
      /*
      判断上传文件至非公开数据库或者公开数据库
      为true则会上传至公开数据库
      */
      isPublic: {
        type: Boolean,
        required: false,
        default: false
      },
      /*
      根据不同需求情况来改变上传按钮的外观
      */
      icon: {
        type: String,
        default: 'icon-上传附件（图）-01'
      },
      /*
      当该组件多次调用时通过该参数来判断当前是第几个组件执行功能，
      默认只有一个，可根据需求不同自定义index的值
      */
      index: {
        type: String,
        required: false,
        default: '0'
      },
      /*
      上传文件大小限制，默认为2M，
      可根据需求不同自定义参数大小来自由选择文件上传大小限制
      */
      fileSize: {
        type: String,
        required: false,
        default: '2'
      }
    },
    /*
    一些执行具体功能的方法
    */
    methods: {
      clickToUpload: function () {
        document.getElementById('upload_file' + this.index).click()
      },
      fileSubmitAuto: function () {
        this.$http.get(/*对应的获取token的后台接口*/,
          {params: {
            isPublic: this.isPublic
          }}).then((response) => {
            if (!response.ok) {
              window.alert('获取token失败:' + response.data.message)
              return
            }
            return response.bodyText
          }).then((token) => {
            if (!token) {
              console.error('获取token失败')
            } else {
              this.token = token
              let file = document.getElementById('upload_file' + this.index)
              let filePath = file.value
              /*这里加filePath === '' 的目的是防止用户误操作，当点击取消时依旧可以上传文件，但是显示报错*/
              if (filePath === '') {
                console.log('当点击取消时执行这里')
              }
              if (filePath !== '') {
                let fileObj = document.getElementById('upload_file' + this.index).files[0]
                let myFileSize = fileObj.size
                if (myFileSize >= this.fileSize * 1024 * 1024) {
                  console.log(this.fileSize)
                  this.filesTooBig()
                } else if (myFileSize <= this.fileSize * 1024 * 1024) {
                  this.fileKey = uuidV1()
                  this.uploadByVueResource(this.fileKey, this.token, fileObj)
                }
              }
            }
          }).catch((error) => {
            console.error(error)
            window.alert('获取token失败')
          })
      },
      uploadByVueResource: function (key, token, fileObj) {
        let formData = new window.FormData()
        if (key !== null && key !== undefined) {
          formData.append('key', key)
          formData.append('token', token)
          formData.append('file', fileObj)
        }
        this.$http.post('/*上传到的云空间地址*/', formData, {
          credentials: false,
          headers: {
            'Content-Type': 'multipart/form-data'
          }
        }).then((response) => {
          let myFileKey = document.getElementById('key' + this.index).value
          let fileObj = document.getElementById('upload_file' + this.index).files[0]
          let myFileName = fileObj.name
          let myFileSize = fileObj.size
          this.$http.get(/*获取下载链接的相应接口，下载链接可以使上传的文件显示出来*/,
            {
              params: {
                fileKey: myFileKey,
                isPublic: this.isPublic
              }}).then((response) => {
                if (!response.ok) {
                  window.alert(response.data.message)
                  return
                } else {
                  return response.bodyText
                }
              }).then((downloadUrl) => {
                /*将文件的一些值与父级进行通讯以便父级可以根据这些值进行相应的操作*/
                this.onSuccess(downloadUrl, myFileKey, myFileSize, myFileName, this.index)
              }).catch((error) => {
                console.error(error)
              })
        })
      },
      onSuccess: function (downloadUrl, myFileKey, myFileSize, myFileName, index) {
        if (!this.disabled) {
          this.$emit('onSuccess', downloadUrl, myFileKey, myFileSize, myFileName, this.index)
          this.filesUploadSuccess()
        } else if (this.disabled) {
          this.filesUploadFailed()
        }
      },
      filesTooBig: function () {
        this.$message({
          showClose: true,
          message: '您上传的文件大小不符合要求，请重新选择',
          type: 'warning',
          duration: '2500'
        })
      },
      filesUploadSuccess: function () {
        this.$message({
          showClose: true,
          message: '上传成功',
          duration: '2500'
        })
      },
      filesUploadFailed: function () {
        this.$message({
          showClose: true,
          message: '上传失败，请重新上传',
          type: 'error',
          duration: '2500'
        })
      }
    }
  }
</script>

<style>
@import "../../theme/index.less";
  .file_btn{
    display:none;
  }
  .submit_btn{
    display:none;
  }
  .click_area{
    display: inline-block;
    width: 10px;
    height: 10px;
    margin-right: 18px;
    text-align: center;
  }
</style>
