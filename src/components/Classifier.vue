<template>
  <div>
    <h1>Step 1: select the input</h1>
    <div class="input-options">
      <input
        type="radio"
        id="sample-input"
        v-model="photoInputType"
        value="sample"
      />
      <div>
        This sample photo of a kangaroo during the day<br />
        <label for="sample-input">
          <img
            alt="sample image"
            class="not-huge-image"
            :class="{ 'not-selected': photoInputType !== 'sample' }"
            :src="base64Prefix + samplePhotoBase64"
            ref="sampleImage1"
          />
        </label>
      </div>
    </div>
    <div class="input-options">
      <input
        type="radio"
        id="file-input"
        v-model="photoInputType"
        value="file"
      />
      <div>
        <label for="file-input">
          Select images:
          <input
            type="file"
            ref="inputPhotos"
            :disabled="photoInputType !== 'file'"
            @change="onFileChange"
            multiple
          />
        </label>
      </div>
    </div>
    <h1>Step 2: run the classifier</h1>
    <button @click="doClassify">Classify {{ inputTypeMsg }}</button>
    {{ runStatus }}
    <div>
      Currently processing image:<br />
      <img ref="processingImage" class="processing-image" />
    </div>
    <h1>Results</h1>
    <div v-if="!isResults">No result (yet)</div>
    <div v-if="isResults">
      <div v-for="currCat of classificationCategories" :key="currCat.name">
        <div class="category-header">
          {{ currCat.name }} ({{ currCat.items.length }} photos)
        </div>
        <div v-if="currCat.items.length === 0">No photos</div>
        <div class="result-photos">
          <div
            v-for="currItem of currCat.items"
            :key="currItem.source"
            class="result-photo-item"
          >
            <img :src="base64Prefix + currItem.source" />
            <div>{{ currItem.result.confidence }}</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import ml5 from 'ml5'

export default {
  name: 'Classifier',
  data() {
    return {
      runStatus: null,
      isModelReady: false,
      classifier: null,
      classifyResult: [],
      photoInputType: 'sample',
      samplePhotoBase64: null,
      base64Prefix: 'data:image/jpeg;base64,',
      selectedFileCount: 0,
    }
  },
  created() {
    this.classifier = ml5.imageClassifier(
      '/ml-model/model.json',
      () => (this.isModelReady = true),
    )
    this.fetchSamplePhoto()
  },
  computed: {
    inputTypeMsg() {
      const type = this.photoInputType
      if (type === 'sample') {
        return 'sample photo'
      }
      if (type === 'file') {
        const count = this.selectedFileCount
        return `${count} local file(s)`
      }
      throw new Error(`Unhandled photoInputType=${type}`)
    },
    isResults() {
      return (this.classifyResult || []).length
    },
    classificationCategories() {
      if (!this.isResults) {
        return []
      }
      const crystalisedResults = this.classifyResult.map(curr => {
        const executiveDecisionOnClassification = curr.confidences.sort(
          highestConfidenceFirst,
        )[0]
        return {
          ...curr,
          result: executiveDecisionOnClassification,
        }
      })
      // FIXME if confidence is below some threshold, we should NOT trust it.
      // Perhaps throw to an "other" category?
      const sortedCrystalisedResults = crystalisedResults.reduce(
        (accum, curr) => {
          const key = curr.result.label
          const existingCollection = accum[key] || []
          existingCollection.push(curr)
          accum[key] = existingCollection
          return accum
        },
        {},
      )
      return this.classifyResult[0].confidences.map(e => ({
        name: e.label,
        items: sortedCrystalisedResults[e.label] || [],
      }))
    },
  },
  methods: {
    onFileChange() {
      this.selectedFileCount = this.$refs.inputPhotos.files.length
    },
    async doClassify() {
      this.classifyResult = []
      this.runStatus = 'processing...'
      if (!this.isModelReady) {
        alert('Model is not ready, cannot run classifier yet')
        return
      }
      const strategy = this[this.photoInputType + 'Strategy']
      try {
        await strategy()
        this.runStatus = 'classification done'
      } catch (err) {
        console.error('Failed to classify', err)
        this.classifyResult = err
      }
    },
    setProcessingPhoto(base64Data) {
      this.$refs.processingImage.src = `${this.base64Prefix}${base64Data}`
    },
    clearProcessingPhoto() {
      this.$refs.processingImage.src = ''
    },
    async sampleStrategy() {
      this.setProcessingPhoto(this.samplePhotoBase64)
      const theImg = this.$refs.processingImage
      this.classifyResult = [
        {
          source: this.samplePhotoBase64,
          confidences: await this.promisedClassify(theImg),
        },
      ]
      this.clearProcessingPhoto()
    },
    async fileStrategy() {
      const theInput = this.$refs.inputPhotos
      const result = []
      for (const curr of theInput.files) {
        const buffer = await curr.arrayBuffer()
        const currBase64 = arrayBufferToBase64(buffer)
        this.setProcessingPhoto(currBase64)
        const currResult = await this.promisedClassify(
          this.$refs.processingImage,
        )
        result.push({
          source: currBase64,
          confidences: currResult,
        })
      }
      this.clearProcessingPhoto()
      this.classifyResult = result
    },
    promisedClassify(inputImage) {
      return new Promise((resolve, reject) => {
        this.classifier.classify(inputImage, (err, result) => {
          if (err) {
            return reject(err)
          }
          return resolve(result)
        })
      })
    },
    async fetchSamplePhoto() {
      const rawResp = await fetch('/sample-images/IMG_0196.JPG')
      const buffer = await rawResp.arrayBuffer()
      this.samplePhotoBase64 = arrayBufferToBase64(buffer)
    },
  },
}

function highestConfidenceFirst(a, b) {
  if (a.confidence < b.confidence) return 1
  if (a.confidence > b.confidence) return -1
  return 0
}

// thanks https://medium.com/front-end-weekly/fetching-images-with-the-fetch-api-fb8761ed27b2
function arrayBufferToBase64(buffer) {
  var binary = ''
  var bytes = [].slice.call(new Uint8Array(buffer))
  bytes.forEach(b => (binary += String.fromCharCode(b)))
  return window.btoa(binary)
}
</script>

<style scoped lang="scss">
.not-huge-image {
  max-width: 20em;
}

.input-options {
  display: flex;

  input {
    flex-grow: 1;
  }

  div {
    flex-grow: 3;
  }
}

img.not-selected {
  filter: grayscale(1);
}

.result-photos {
  display: flex;
  flex-wrap: wrap;

  .result-photo-item {
    max-width: 10em;
    margin: 5px;

    img {
      width: 100%;
    }
  }
}

.category-header {
  background-color: #ddd;
  margin-top: 1em;
}

.processing-image {
  max-width: 10em;
}
</style>
