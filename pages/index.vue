<template>
  <section>
    <h1>Figma Slides</h1>
    <h2>Turn your Figma frames into Google slides</h2>

    <template v-if="signedIn">
      <template v-if="creating">
        <template v-if="done">
          <p>Your presentation has been created.</p>
        </template>
        <template v-else>
          <p>We are now creating your presentation. This could take a moment depending on the quantity and complexity of frames.</p>
        </template>
      </template>
      <template v-else>
        <ol>
          <li>Create a Figma presentation with one page consisting of 1920 x 1080 frames for each slide. Here's an <a href="https://www.figma.com/file/MpVhHAInw5db1y9RSw3S1Tdf/Presentation-Example?node-id=0%3A1">example</a>.</li>
          <li><a href="https://www.figma.com/developers/docs#access-tokens" target="_blank">Generate a personal access token</a> and paste it below.</li>
          <li>Copy the file key from your Figma file url and paste it below. https://www.figma.com/file/<strong>FILE_KEY</strong>/File-Name</li>
          <li>Click "Create Presentation" and wait a moment</li>
        </ol>
        <p><input type="password" v-model="figmaAccessToken" placeholder="Figma access token" /></p>
        <p><input type="text" v-model="figmaFileKey" placeholder="Figma file key" /></p>
        <p><button @click="createPresentation">Create Presentation</button></p>
      </template>
      <p><button @click="deauthorize">Logout</button></p>
    </template>
    <template v-else>
      <p>This simple app can turn a Figma design file's frames into slides of a newly created Google Slides presentation. Please login with your Google account to get started.</p>
      <p><button @click="authorize">Login with Google Acccount</button></p>
    </template>
  </section>
</template>

<script>
import axios from 'axios'

export default{
  data() {
    return {
      signedIn: false,
      figmaAccessToken: '',
      figmaFileKey: '',
      creating: false,
      done: false
    }
  },
  head: {
    title: 'Figma Slides',
    meta: [
      {
        property: 'og:site_name',
        content: 'Figma Slides'
      }, {
        property: 'og:image',
        content: 'https://www.figmaslides.app/images/social.jpg'
      }, {
        property: 'og:title',
        content: 'Figma Slides'
      }, {
        property: 'og:description',
        content: 'Turn your Figma frames into Google Slides.'
      }, {
        property: 'og:url',
        content: 'https://www.figmaslides.app'
      }, {
        property: 'og:type',
        content: 'website'
      }, {
        name: 'twitter:card',
        content: 'summary_large_image'
      }, {
        name: 'twitter:site',
        content: '@leemartin'
      }, {
        name: 'twitter:title',
        content: 'Figma Slides'
      }, {
        name: 'twitter:description',
        content: 'Turn your Figma frames into Google Slides.'
      }, {
        name: 'twitter:image',
        content: 'https://www.figmaslides.app/images/social.jpg'
      }, {
        name: 'twitter:url',
        content: 'https://www.figmaslides.app'
      }
    ],
    script: [
      {
        src: 'https://apis.google.com/js/api.js'
      }
    ]
  },
  methods: {
    start() {
      gapi.client.init({
        apiKey: process.env.googleApiKey,
        clientId: process.env.googleClientId,
        discoveryDocs: ["https://slides.googleapis.com/$discovery/rest?version=v1"],
        scope: 'https://www.googleapis.com/auth/presentations'
      }).then(() => {
        gapi.auth2.getAuthInstance().isSignedIn.listen(this.updateSigninStatus)

        this.updateSigninStatus(gapi.auth2.getAuthInstance().isSignedIn.get())
      }, (error) => {
        console.error(error)
      })
    },
    authorize() {
      gapi.auth2.getAuthInstance().signIn()
    },
    deauthorize() {
      gapi.auth2.getAuthInstance().signOut()
    },
    updateSigninStatus(isSignedIn) {
      this.signedIn = isSignedIn
    },
    createPresentation() {
      this.creating = true

      let figma = axios.create({
        baseURL: 'https://api.figma.com',
        headers: {
          'X-FIGMA-TOKEN': this.figmaAccessToken
        }
      })

      figma.get(`/v1/files/${this.figmaFileKey}`)
      .then((response) => {
        let data   = response.data
        let title  = data.name
        let page   = data.document.children[0]
        let slides = page.children
        let ids    = slides.map(slide => slide.id)

        figma.get(`/v1/images/${this.figmaFileKey}?ids=${ids.join(',')}`)
        .then((response) => {
          let data   = response.data
          let images = data.images

          gapi.client.slides.presentations.create({
            title: title,
            pageSize: {
              width: {
                magnitude: 1920,
                unit: 'PT'
              },
              height: {
                magnitude: 1080,
                unit: 'PT'
              }
            }
          }).then((response) => {
            let presentationId = response.result.presentationId

            let requests = Object.keys(images).map((key, index) => {
              return [
                {
                  createSlide: {
                    objectId: `slide${key}`,
                    slideLayoutReference: {
                      predefinedLayout: 'BLANK'
                    }
                  }
                }, {
                  updatePageProperties: {
                    objectId: `slide${key}`,
                    pageProperties: {
                      pageBackgroundFill: {
                        stretchedPictureFill: {
                          contentUrl: images[key]
                        }
                      }
                    },
                    fields: "pageBackgroundFill"
                  }
                }
              ]
            })

            requests = [].concat(...requests)

            requests.unshift({
              deleteObject: {
                objectId: 'p'
              }
            })

            gapi.client.slides.presentations.batchUpdate({
              presentationId: presentationId,
              requests: requests
            }).then((response) => {
              this.done = true
            }).catch(console.error)
          })
        })
      })
    }
  },
  mounted() {
    gapi.load('client', this.start)
  }
}
</script>
