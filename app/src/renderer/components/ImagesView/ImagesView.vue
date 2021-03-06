<template>
  <div>
    <Button type="primary" icon="refresh" @click="refreshImages">Refresh</Button>
    <Button type="primary" icon="plus-round" @click="imagePullModal = true">Pull</Button>
    <Modal v-model="imagePullModal" title="Pull Image" @on-ok="pullImage" @on-cancel="repoTag = ''">
      <Input v-model="repoTag" placeholder="Image Name (and Tag)"></Input>
    </Modal>
    <div class="docker-hub-panel">
      <login-panel></login-panel>
    </div>
    <br><br>
    <div v-if="hasFoundImages">
      <Card v-for="image in images" class="image-card">
        <p slot="title" class="image-card-title">
          <Tooltip placement="right">
            {{getImageName(image.RepoTags[0])}}
            <div slot="content" class="description-pop">
              <p v-for="name in image.RepoTags">{{name}}</p>
            </div>
          </Tooltip>
        </p>
        <p>
          Tags: <Tag v-for="tag in getTags(image.RepoTags)">{{tag}}</Tag>
        </p>
        <p>Size: {{formatBytes(image.Size)}}</p>
        <p>Created: {{getDateTime(image.Created)}}</p>
        <Button type="primary" @click="inspectImage(image.Id)">Inspect</Button>
        <image-control-panel class="control-panel" :image-id="image.Id"
            @input="function (newData) { loadImages() }"
            @image-removed="function (removed) { loadImages() }">
        </image-control-panel>
      </Card>
    </div>
    <div v-else>
      <pre>{{error}}</pre>
    </div>
    <foot-logs-view v-model="footLogs"></foot-logs-view>
  </div>
</template>

<script>
  import ImageControlPanel from './ImageControlPanel'
  import FootLogsView from '../FootLogsView'
  import LoginPanel from '../DockerHubView/LoginPanel'

  import docker from '../../js/docker'
  import notify from '../../js/notify'
  import notNull from '../../js/notNull'
  import parseRepoTag from '../../js/parseRepoTag'
  import formatBytes from '../../js/formatBytes'

  export default {
    components: {
      ImageControlPanel,
      FootLogsView,
      LoginPanel
    },
    data () {
      return {
        images: [],
        hasFoundImages: false,
        error: {},
        imagePullModal: false,
        repoTag: '',
        footLogs: {}
      }
    },
    watch: {
      images: function (newImages) {
        this.hasFoundImages = notNull(newImages) && newImages.length > 0
      }
    },
    methods: {
      refreshImages () {
        this.loadImages()
        if (notNull(this.error) && this.error !== {}) {
          notify('Refreshed: ' + this.images.length + ' images found!')
        }
      },
      pullImage () {
        var self = this

        this.$set(self.footLogs, 'pullLog', '')

        function imagePulled (stream) {
          function onFinished (err, output) {
            self.$delete(self.footLogs, 'pullLog')
            if (err) {
              notify(err)
              return
            }
            notify('New image is pulled!')
          }

          function onProgress (event) {
            self.$set(self.footLogs, 'pullLog', JSON.stringify(event))
          }

          docker.modem.followProgress(stream, onFinished, onProgress)
        }

        docker.pull(this.repoTag)
          .then(imagePulled)
          .catch(notify)
      },
      inspectImage (imageId) {
        this.$router.push({
          name: 'single-image-view',
          params: { imageId: imageId }
        })
      },
      loadImages () {
        var self = this

        var queries = {
          all: false
        }

        function updateImages (images) {
          self.images = images
          self.error = {}
        }

        function updateErrored (err) {
          self.images = []
          self.error = err
          notify(err)
        }

        docker.listImages(queries)
          .then(updateImages)
          .catch(updateErrored)
      },
      getImageName (repoTag) {
        return parseRepoTag(repoTag).repository
      },
      getTag (repoTag) {
        return parseRepoTag(repoTag).tag
      },
      getTags (repoTags) {
        return repoTags.map(function (repoTag) {
          return parseRepoTag(repoTag).tag
        })
      },
      getDateTime (seconds) {
        return new Date(seconds * 1000).toLocaleString()
      },
      formatBytes: formatBytes
    },
    created () {
      console.log('auth.token: ', this.$store.state.auth.token)
      console.log('user.username: ', this.$store.state.user.username)
      this.loadImages()
    }
  }
</script>

<style scoped>
  .docker-hub-panel {
    display: inline-block;
    float: right;
    width: 350px;
  }

  .image-card {
    width: 300px;
    display: inline-block;
    margin: 5px 5px;
  }

  .image-card-title {
    height: 26px;
  }

  .control-panel {
    display: inline-block;
  }

  .description-pop p {
    display: block;
    white-space: normal;
    color: #ffffff;
  }

  .description-pop {
    white-space: normal;
  }
</style>
