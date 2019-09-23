<template>
  <div>
    <slot v-for="upload in currentUploads" :upload="upload" :$pause="pauseFactory(upload.id)" :$resume="startFactory(upload.id)">
    </slot>
  </div>
</template>

<script>
require('spark-md5')
require('q')
require('cloud-storage-resume-signed')

function debounce(func, wait, immediate) {
	var timeout
	return function() {
		var context = this, args = arguments
		var later = function() {
			timeout = null
			if (!immediate) func.apply(context, args)
		}
		var callNow = immediate && !timeout
		clearTimeout(timeout)
		timeout = setTimeout(later, wait)
		if (callNow) func.apply(context, args)
	}
}

let disconnectedF = null
let lastSentProgress = 0
let totalProgressF = null

const defaultTimeout = 5
let timeoutInterval = null

const status = {
  NOT_STARTED: 'NOT_STARTED',
  STARTED: 'STARTED',
  PAUSED: 'PAUSED',
  FINISHED: 'FINISHED',
  ERROR: 'ERROR'
}

const errorTypes = {
  DISCONNECTED: 'DISCONNECTED',
  URI_EXPIRED: 'URI_EXPIRED',
  UNKKOW_ERROR: 'UNKKOW_ERROR'
}

export default {
  name: 'gcs-signed-uploader',
  props: [
    'files',
    'timeout',
    'mock'
  ],
  data () {
    return {
      'currentUploads': [

      ],
      'managers': {}
    }
  },
  mounted () {
    const self = this

    disconnectedF = debounce(function () {
      self.$emit('onConnectionError', self.retryContinue)
    }, self.timeout ? (self.timeout * 1000) : 5000, true)

    let intervalSendLastTotalProgress = null
    totalProgressF = function (internal) {
      let totalUploadSize = self.currentUploads.reduce((ac, at) => {
        return ac + at.size
      }, 0)
      let totalUploaded = self.currentUploads.reduce((ac, at) => {
        return ac + (at.size * at.progress)
      }, 0)

      if (intervalSendLastTotalProgress !== null) {
        clearTimeout(intervalSendLastTotalProgress)
      }
      intervalSendLastTotalProgress = window.setTimeout(() => {
        totalProgressF(true)
      }, 2000)
      if (internal) {
        clearTimeout(intervalSendLastTotalProgress)
      }
      self.$emit('progress', totalUploaded / totalUploadSize)
      lastSentProgress = (new Date()).getTime()
    }

    timeoutInterval = window.setInterval(() => {
      self.currentUploads.forEach(u => {
        if (self.mock !== undefined && u.lastUploadProgress < ((new Date()).getTime() - (defaultTimeout * 1000)) && u.status === status.STARTED) {
         managers[u.id].setDisconnected()
        }
      })
    }, 2000)

    if (this.mock !== undefined) {
      this.currentUploads = this.mock
    }
  },
  destroyed () {
    window.clearInterval(timeoutInterval)
  },
  watch: {
    files () {
      const self = this
      const newFile = this.fileAdded()

      let manager = null
      let startingError = null
      if (newFile) {
        const added = this.files.filter(f => f.id === newFile.id)[0]

        try {
          manager = new cloudStorageSignedResumer.Uploader({
            'file': added.file,
            'signedURI': added.signedURI
          })
        } catch (e) {
          console.log(e)
          startingError = e
        }

        const newUpload = {
          'id': added.id,
          'autostarted': added.autostart,
          'progress': 0,
          'lastUploadProgress': (new Date()).getTime,
          'status': startingError ? status.ERROR : status.NOT_STARTED,
          'errorType': startingError || null,
          'filename': added.file.name,
          'size': added.file.size
        }

        this.currentUploads.push(newUpload)

        if (!manager) {
          return
        }

        this.managers[added.id] = manager
        
        manager.on('progress', (progress) => {
          newUpload.progress = progress
          newUpload.lastUploadProgress = (new Date()).getTime()
          if ((newUpload.lastUploadProgress - lastSentProgress > 1000)) {
            totalProgressF(false)
          }
        })

        manager.on('uploadstarted', () => {
          newUpload.status = status.STARTED
          newUpload.errorType = null
        })

        manager.on('uploadpaused', () => {
          newUpload.status = status.PAUSED
          self.$emit('onUploadPaused', newFile.id)
        })

        manager.on('uploadfinished', () => {
          newUpload.status = status.FINISHED
          self.$emit('onUploadFinished', newFile.id)
        })

        manager.on('disconnected', () => {
          newUpload.status = status.ERROR
          newUpload.errorType = errorTypes.DISCONNECTED
          self.disconected()
        })

        manager.on('uploaderror', (e) => {
          self.$emit('onUploadError', newFile.id)
          newUpload.status = status.ERROR
          newUpload.errorType = errorTypes.UNKKOW_ERROR
        })

        if (added.autostart) {
          manager.upload()
        }
      }
    }
  },
  methods: {
    disconected () {
      disconnectedF(this)
    },
    pauseFactory (id) {
      const self = this
      return function () {
        self.managers[id].pause()
      }
    },
    startFactory (id) {
      const self = this
      return function () {
        self.managers[id].upload()
      }
    },
    fileRemoved () {
      for (let i = 0; i < this.currentUploads.length; i++) {
        let removed = true
        for (let j = 0; j < this.files.length; j++) {
          if (this.files[j].id === this.currentUploads[i].id) {
            removed = false
          }
        }
        if (removed) {
          return {
            'id': this.files[i].id
          }
        }
      }
    },
    fileAdded () {
      for (let i = 0; i < this.files.length; i++) {
        let added = true
        for (let j = 0; j < this.currentUploads.length; j++) {
          if (this.files[i].id === this.currentUploads[j].id) {
            added = false
          }
        }
        if (added) {
          return {
            'id': this.files[i].id
          }
        }
      }
    },
    retryContinue () {
      this.currentUploads.forEach(u => {
        if (u.status === status.ERROR && u.errorType === errorTypes.DISCONNECTED) {
          this.managers[u.id].upload()
        }
      })
    }
  }
}
</script>

<style>

</style>
