<template>
  <uploader
    :options="options"
    :file-status-text="fileStatusText"
    class="uploader-example"
    ref="uploader"
    @file-complete="fileComplete"
    @complete="complete"
    :autoStart="false"
    @file-added="onFileAdded">
    <uploader-btn class="uploader--btn">
      <uploader-drop class="uploader--drop">
        <p>Drop files here to upload or</p>
      </uploader-drop>
    </uploader-btn>
    <uploader-files>
      <template slot-scope="{ files }">
        <ul>
          <li v-for="file in files" :key="file.id">
            <uploader-file :class="`file_${file.id}`" class="customStatus uploader-file" :file="file">
              <template slot-scope="{ progressStyle }">
                <div class="file">
                  <div class="btns">
                    <span @click="pause(file)">暂停</span>
                    <span @click="resume(file)">继续</span>
                    <span @click="retry(file)">重试</span>
                    <span @click="remove(file)">取消</span>
                  </div>
                  <div class="progress--bg">
                    <div class="progress--bar" :style="progressStyle"></div>
                  </div>
                </div>
              </template>
            </uploader-file>
          </li>
        </ul>
      </template>
    </uploader-files>
  </uploader>
</template>

<script>
export default {
  data() {
    return {
      options: {
        target: '//localhost:3000/upload',
        testChunks: false,
      },
      // fileStatusText: {
      //   success: '成功了',
      //   error: '出错了',
      //   uploading: '上传中',
      //   paused: '暂停中',
      //   waiting: '等待中',
      // },
      fileStatusText: (status, response) => {
        // const statusTextMap = {
        //   uploading: 'uploading',
        //   paused: 'paused',
        //   waiting: 'waiting',
        // }
        // if (status === 'success' || status === 'error') {
        //   // return response data ?
        //   return response.data
        // } else {
        //   return statusTextMap[status]
        // }
        console.log(status)
      },
    }
  },
  methods: {
    complete() {
      console.log('complete', arguments)
    },
    fileComplete() {
      console.log('file complete', arguments)
    },
    onFileAdded(file) {
      // this.computeMD5(file).then(result => this.startUpload(result))
    },
    computeMD5(file) {
      let fileReader = new FileReader()
      let time = new Date().getTime()
      let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
      let currentChunk = 0
      const chunkSize = 10 * 1024 * 1000
      let chunks = Math.ceil(file.size / chunkSize)
      let spark = new SparkMD5.ArrayBuffer()

      // 文件状态设为"计算MD5"
      this.statusSet(file.id, 'md5')

      loadNext()

      return new Promise((resolve, reject) => {
        fileReader.onload = e => {
          spark.append(e.target.result)

          if (currentChunk < chunks) {
            currentChunk++
            loadNext()

            // 实时展示MD5的计算进度
            this.$nextTick(() => {
              const md5ProgressText = '校验MD5 ' + ((currentChunk / chunks) * 100).toFixed(0) + '%'
              document.querySelector(`.custom-status-${file.id}`).innerText = md5ProgressText
            })
          } else {
            let md5 = spark.end()

            // md5计算完毕
            resolve({ md5, file })

            console.log(`MD5计算完毕：${file.name} \nMD5：${md5} \n分片：${chunks} 大小:${file.size} 用时：${new Date().getTime() - time} ms`)
          }
        }

        fileReader.onerror = function () {
          this.error(`文件${file.name}读取出错，请检查该文件`)
          file.cancel()
          reject()
        }
      })

      function loadNext() {
        let start = currentChunk * chunkSize
        let end = start + chunkSize >= file.size ? file.size : start + chunkSize

        fileReader.readAsArrayBuffer(blobSlice.call(file.file, start, end))
      }
    },

    resume(file) {
      file.resume()
    },
    pause(file) {
      file.pause()
    },
    retry(file) {
      file.retry()
    },
    remove(file) {
      file.remove()
    },
  },
  computed: {
    uploader() {
      return this.$refs.uploader.uploader
    },
  },
}
</script>

<style>
.uploader-example {
  width: 880px;
  padding: 15px;
  margin: 40px auto 0;
  font-size: 12px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.4);
}
.uploader-example .uploader-btn {
  margin-right: 4px;
}
.uploader-example .uploader-list {
  max-height: 440px;
  overflow: auto;
  overflow-x: hidden;
  overflow-y: auto;
}

.uploader--btn {
  width: 100%;
}

.progress--bg {
  height: 8px;
  border-radius: 4px;
  background-color: grey;
  position: relative;
  overflow: hidden;
}
.progress--bar {
  position: absolute;
  border-radius: 4px;
  background-color: red;
  width: 100%;
  height: 100%;
}

.customStatus {
  height: auto !important;
  line-height: normal !important;
}
</style>
