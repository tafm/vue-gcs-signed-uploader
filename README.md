Vue GCS resume signed
==========

This is not a 'visual' component, but a upload manager to build upload interface on top of it (its more like a data-table, where each row its a upload)


Example:
-
```html
<template>
	<div>
		<input  id="myfile"  type="file">
		<input  id="mysigneduri"  type="text">
		<Uploader  :files="files">
			<div  slot-scope="{upload, $pause, $resume}">
				{{ upload }}
				<div  @click="$pause()"  style="width: 50px; height: 50px; background-color: red;"></div>
				<div  @click="$resume()"  style="width: 50px; height: 50px; background-color: green;"></div>
			</div>
		</Uploader>
	</div>
</template>

<script>

import  Uploader  from  'vue-gcs-signed-uploader'
export  default {
	data () {
		return {
			files: []
		}
	},
	name:  'MyComponent',
	mounted () {
		let  self  =  this
		document.getElementById('myfile').onchange  =  function (e) {
			self.addFile(this.files[0])
		}
	},
	components: {
		Uploader
	},
	methods: {
		addFile (file) {
			let  id  =  Math.ceil(Math.random() *  100)
			this.files.push({
				'file':  file,
				'id':  id,
				'autostart':  true,
				'signedURI': document.getElementById('mysigneduri').value
			})
		}
	}
}

</script>

  

<style  scoped>
</style>
```
To start a upload just push to the files array and given a `id` to identify what file was added/removed

Events:
-
*onUploadPaused*
*onUploadFinished*
*onUploadError*
*onConnectionError*

Status:
-
*NOT_STARTED*
*STARTED*
*PAUSED*
*ERROR*

Error types:
-
*URI_EXPIRED*
*DISCONNECTED*

Mocking
-
Use the `:mock` to send a list of uploads for design purpose

```html
<template>
	<Uploader :mock="mock">
		...
	</Uploader>
</template>
<script>
...
data() {
	return {
		mock: [
			{
				'id': 1,
				'status': 'STARTED',
				'progress': 0.5,
				'autostarted': true,
				'errorType': null
			}
		]
	}
}
</script>
```