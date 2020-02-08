<template>
  <div>
    <div>
      <img
        alt="sample image"
        class="not-huge-image"
        src="/sample-images/IMG_0196.JPG"
        ref="sampleImage1"
      />
    </div>
    <button @click="doClassify">Classify</button> {{ runStatus }}
    <p>Most likely result: {{ mostLikely }}</p>
    <div>Other, less likely, results:</div>
    <ul>
      <li v-for="curr of notLikely" :key="curr.label">
        {{ curr.label }} ({{ curr.confidence }})
      </li>
    </ul>
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
      classifyResult: null,
    }
  },
  created() {
    this.classifier = ml5.imageClassifier(
      '/ml-model/model.json',
      () => (this.isModelReady = true),
    )
  },
  computed: {
    sortedHighestConfidenceFirst() {
      return (this.classifyResult || []).sort(highestConfidenceFirst)
    },
    mostLikely() {
      return (this.sortedHighestConfidenceFirst[0] || { label: 'nothing' })
        .label
    },
    notLikely() {
      return this.sortedHighestConfidenceFirst.slice(1)
    },
  },
  methods: {
    doClassify() {
      this.runStatus = 'processing...'
      if (!this.isModelReady) {
        alert('Model is not ready, cannot run classifier yet')
        return
      }
      const theImg = this.$refs.sampleImage1
      this.classifier.classify(theImg, (err, result) => {
        if (err) {
          console.error('Failed to classify', err)
          this.classifyResult = err
          return
        }
        this.classifyResult = result
        this.runStatus = 'classification done'
      })
    },
  },
}

function highestConfidenceFirst(a, b) {
  if (a.confidence < b.confidence) return 1
  if (a.confidence > b.confidence) return -1
  return 0
}
</script>

<style scoped lang="scss">
.not-huge-image {
  max-width: 20em;
}
</style>
