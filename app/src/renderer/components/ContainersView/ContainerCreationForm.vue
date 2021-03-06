<template>
  <div>
    <Form :model="defaultSettings" :label-width="80">
      <Form-item label="Image" required>
        <Input v-model="defaultSettings.Image" placeholder="Image Name"></Input>
      </Form-item>
      <Form-item label="Name">
        <Input v-model="defaultSettings.name" placeholder="New name of your container"></Input>
      </Form-item>
    </Form>
    <Button class="import-button" type="primary" @click="openFileDialog">
      Import from JSON
    </Button>
    <json-form class="advanced-settings-form" name="Advanced Settings" :label-width="80"
        v-model="advancedSettings">
    </json-form>
  </div>
</template>

<script>
  import JsonForm from '../JsonForm/JsonForm'

  import { ipcRenderer } from 'electron'
  import docker from '../../js/docker'
  import notify from '../../js/notify'
  import jsonFileImportInit from '../../js/jsonFileImportInit'

  export default {
    components: {
      JsonForm
    },
    props: {
      value: {
        type: String,
        default: ''
      }
    },
    data () {
      return {
        defaultSettings: {
          Image: this.value,
          name: ''
        },
        importedSettings: {},
        advancedSettings: {}
      }
    },
    watch: {
      value: function (newValue) {
        this.defaultSettings.Image = newValue
      }
    },
    methods: {
      submit () {
        var self = this

        function containerCreated (container) {
          notify('New container ID ' + container.id +
                 ' created from image ' + self.defaultSettings.Image + ' !')
          self.$emit('container-created', container)
        }

        function creationErrored (err) {
          notify(err)
          self.$emit('container-creation-errored', err)
        }

        docker.createContainer(Object.assign({}, this.defaultSettings, this.advancedSettings))
          .then(containerCreated)
          .catch(creationErrored)

        this.reset()
      },
      reset () {
        this.defaultSettings = {
          Image: '',
          name: ''
        }
        this.advancedSettings = this.importedSettings
      },
      openFileDialog () {
        ipcRenderer.send('open-file-dialog')
      }
    },
    created () {
      var self = this
      this.stringifiedSettings = JSON.stringify(this.importedSettings, null, 4)

      function readFileCB (err, data) {
        if (err) {
          notify(err)
        }

        var parsedJSON = JSON.parse(data)
        self.importedSettings = parsedJSON
        self.advancedSettings = self.importedSettings
      }

      jsonFileImportInit(readFileCB)
    }
  }
</script>

<style scoped>
  .switch {
    display: inline-block;
  }
  .import-button {
    display: inline-block;
  }
  .advanced-settings-form {
    margin-top: 10px;
  }
</style>
