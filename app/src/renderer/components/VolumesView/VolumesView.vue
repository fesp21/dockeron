<template>
  <div>
    <Button type="primary" icon="refresh" @click="refreshVolumes">Refresh</Button>
    <br>
    <div v-if="hasFoundVolumes">
      <Card v-for="volume in volumes" class="volume-card">
        <Poptip trigger="hover">
            {{volume.Name}}
          <div slot="content">
            <Button type="primary" @click="inspectVolume(volume.Name)">Inspect</Button>
            <volume-control-panel class="control-panel" :volume-name="volume.Name"
                @input="function (newData) { loadVolumes() }"
                @volume-removed="function (removed) { loadVolumes() }">
            </volume-control-panel>
          </div>
        </Poptip>
        <Tag class="volume-state-tag">{{volume.Driver}}</Tag>
      </Card>
    </div>
    <div v-else>
      <pre>{{error}}</pre>
    </div>
  </div>
</template>

<script>
  import VolumeControlPanel from './VolumeControlPanel'

  import docker from '../../js/docker'
  import notify from '../../js/notify'
  import notNull from '../../js/notNull'
  import formatBytes from '../../js/formatBytes'

  export default {
    components: {
      VolumeControlPanel
    },
    data () {
      return {
        volumes: [],
        hasFoundVolumes: false,
        error: {}
      }
    },
    watch: {
      volumes: function (newVolumes) {
        this.hasFoundVolumes = notNull(newVolumes) && newVolumes.length > 0
      }
    },
    methods: {
      refreshVolumes () {
        this.loadVolumes()
        if (notNull(this.error) && this.error !== {}) {
          notify('Refreshed: ' + this.volumes.length + ' volumes found!')
        }
      },
      inspectVolume (volumeName) {
        this.$router.push({
          name: 'single-volume-view',
          params: { volumeName: volumeName }
        })
      },
      loadVolumes () {
        var self = this

        function updateVolumes (listResults) {
          self.volumes = listResults.Volumes
          self.error = {}
        }

        function updateErrored (err) {
          self.volumes = []
          self.error = err
          notify(err)
        }

        docker.listVolumes()
          .then(updateVolumes)
          .catch(updateErrored)
      },
      formatBytes: formatBytes
    },
    created () {
      this.loadVolumes()
    }
  }
</script>

<style scoped>
  .volume-card {
    /*width: 300px;*/
    width: 100%;
    display: inline-block;
    margin: 5px 5px;
  }

  .volume-state-tag {
    position: absolute;
    right: 3px;
  }

  .control-panel {
    display: inline-block;
  }
</style>
