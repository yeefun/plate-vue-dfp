<template>
  <div :class="`${className} ${extClass}`" :id="elId" :data-adunit="adunit" :pos="pos" :data-slot-key="slotKey" :style="style" v-if="!isEmpty" :sessionId="sessionId" :size="size"></div>
</template>
<script>
  const debug = require('debug')('CLIENT:Dfp')
  export default {
    computed: {
      elId () {
        return `${this.adunit}-auto-gen-id-${this.suffixId}`
      },
      adunit () {
        let _unitId = !this.unitId
          ? this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'aduid' ]
          : this.unitId
        if (this.ifDevMode) {
          _unitId = `test_${_unitId}`
        }
        return _unitId
      },
      className () {
        return this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'cont-class' ].join(' ')
      },
      currPath () {
        return this.$route.fullPath
      },
      currConfig () {
        return Object.assign({
          dfpUnits: this.dfpUnits,
          section: this.section,
          dfpId: this.dfpId,
          mode: this.mode,
          options: this.options
        }, this.config)
      },
      ifDevMode () {
        return this.config.mode === 'dev'
      },
      isEmpty () {
        return !this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ]
      },
      sessionId () {
        return this.currConfig.sessionId
      },
      slotKey () {
        return `${this.pos}-${this.suffixId}`
      }
    },
    data () {
      return {
        style: ''
      }
    },
    name: 'Dfp',
    methods: {
      defineDfp () {
        googletag.cmd.push(() => {
          // if (window.adSlots[ this.slotKey ] && window.adSlots[ this.slotKey ].adId === this.elId) {
          if (window.adSlots[ this.slotKey ]) {
            googletag.destroySlots([ window.adSlots[ this.pos ] ])
            googletag.pubads().clear([ window.adSlots[ this.pos ] ])
          }
          if (this.isEmpty) { return }

          const slot = document.getElementById(this.elId)
          if (slot) {
            debug(`GOING TO REMOVE SLOT#${this.elId} LAGACY STYLE.`)
            slot.removeAttribute('style')
            slot.innerHtml = ''
            slot.setAttribute('style', this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'cont-style' ].join(';'))
          } else {
            debug(`SLOT #${this.elId} DOESN'T EXIST AT THIS MOMENT.`)
          }
          const isSlotVisible = slot ? (slot.currentStyle ? slot.currentStyle.display : window.getComputedStyle(slot, null).display) : null
          if (isSlotVisible === 'none') {
            delete window.adSlots[ this.slotKey ]
            return
          }

          const isOutOfPage = this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ].outOfPage
          let _s = {}
          if (!isOutOfPage) {
            _s = googletag.defineSlot(`/${this.currConfig.dfpId}/${this.adunit}`
                      , this.getDimensions(this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'dimensions' ])
                      , this.elId)
            try {
              _s.addService(googletag.pubads())
            } catch (err) {
              debug('UNABLED TO RENDER AD', this.elId)
              debug('ERR', err)
              return
            }
            // const mapping = (this.currConfig.options[ 'sizeMapping' ] && this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'size-mapping' ]) ? this.currConfig.options[ 'sizeMapping' ][ this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'size-mapping' ] ] : undefined
            const size = slot && slot.getAttribute('size')
            const mapping = this.currConfig.options[ 'sizeMapping' ]
              ? size
              ? this.currConfig.options[ 'sizeMapping' ][ size ]
              : this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'size-mapping' ]
              ? this.currConfig.options[ 'sizeMapping' ][ this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'size-mapping' ] ]
              : undefined
              : undefined

            debug(`AD ${this.pos}.`)
            debug(`SIZE ${size}.`)
            debug(`SIZE-MAPPING ${this.currConfig.options[ 'sizeMapping' ]}.`)
            debug(`SIZE-MAPPING IN AFPUNITS ${this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'size-mapping' ]}`)

            if (mapping) {
              // Convert verbose to DFP format
              let map = googletag.sizeMapping()
              mapping.forEach((k, v) => {
                map = map.addSize(k.browser, k.ad_sizes)
              })
              _s.defineSizeMapping(map.build())
            }
            window.adSlots[ this.slotKey ] = _s
          } else {
            debug('##### OOP DETECTED #####')
            // _s = googletag.defineOutOfPageSlot(`/${this.currConfig.dfpId}/${this.adunit}`
            //                       , this.elId)
            _s = googletag.defineOutOfPageSlot(`/${this.currConfig.dfpId}/${this.adunit}`, this.elId)
            try {
              _s.addService(googletag.pubads())
            } catch (err) {
              debug('UNABLED TO RENDER OOP AD', this.elId)
              debug('ERR', err)
              return
            }
            // _s = googletag.pubads().defineOutOfPagePassback(`/${this.currConfig.dfpId}/${this.adunit}`)
            window.adSlots[ this.slotKey ] = _s
            window.adSlots[ this.slotKey ].isOutOfPage = true
          }
          window.adSlots[ this.slotKey ].adId = this.elId
          window.adSlots[ this.slotKey ].displayFlag = false
          window.adSlots[ this.slotKey ].refreshFlag = false

          googletag.cmd.push(() => {
            if (!window.adSlots[ this.slotKey ].refreshFlag) {
              googletag.pubads().refresh([ window.adSlots[ this.slotKey ] ])
              // googletag.display(window.adSlots[ this.slotKey ].adId)
              window.adSlots[ this.slotKey ].refreshFlag = true
            }
          })
        })
      },
      getDimensions (dimes) {
        const dimensions = []
        if (typeof dimes !== 'undefined' && dimes !== '') {
          dimes.split(',').forEach((v) => {
            const dimensionSet = v.split('x')
            if (dimensionSet.length > 1) {
              dimensions.push([ parseInt(dimensionSet[ 0 ], 10), parseInt(dimensionSet[ 1 ], 10) ])
            } else {
              if (dimensionSet[ 0 ] === 'fluid') {
                dimensions.push(dimensionSet[0])
              }
            }
          })
        } else {
          dimensions.push([ this.$el.offsetWidth, this.$el.offsetHeight ])
        }
        return dimensions
      },
      isClient () {
        const browser = typeof window !== 'undefined'
        return browser
      }
      // style () {
      //   return this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'cont-style' ].join(';')
      // }
    },
    props: {
      extClass: {
        default: () => { return '' }
      },
      dfpId: {
        default: () => { return '' }
      },
      dfpUnits: {
        default: () => { return '' }
      },
      options: {
        default: () => {
          return {
            dfpID: '',
            setTargeting: {},
            setCategoryExclusion: '',
            setLocation: '',
            enableSingleRequest: true,
            collapseEmptyDivs: 'original',
            refreshExisting: true,
            disablePublisherConsole: false,
            disableInitialLoad: true,
            setCentering: false,
            setForceSafeFrame: false,
            noFetch: false,
            namespace: undefined,
            sizeMapping: undefined,
            afterAdBlocked: undefined,
            afterEachAdLoaded: undefined,
            afterAllAdsLoaded: undefined,
            rendered: 0,
            onloaded: [],
            adUnits: []
          }
        }
      },
      size: {},
      pos: {
        default: () => { return '' }
      },
      section: {
        default: () => { return 'default' }
      },
      unitId: {
        default: () => { return undefined }
      },
      suffixId: {
        type: Number,
        default: 1
      },
      mode: {
        default: () => { return 'prod' }
      },
      config: {
        default: () => {
          return {
            mode: 'prod'
          }
        }
      }
    },
    mounted () {
      debug(`AD ${this.pos} MOUNTED.`)
      if (window && window[ 'googletag' ] && window[ 'googletag' ][ 'apiReady' ] && this.currConfig.firstDfpRender) {
        debug(`AD ${this.pos} IS GONNA DEFINE ITSELF.`)
        this.defineDfp()
      }
      this.style = this.currConfig && !this.isEmpty && this.currConfig.dfpUnits[ this.currConfig.section ][ this.pos ][ 'cont-style' ].join(';')
    },
    watch: {
      currPath: function () {
        if (this.isClient() && window && window[ 'googletag' ] && window[ 'googletag' ][ 'apiReady' ]) {
          debug(`AD ${this.pos} IS GONNA DEFINE ITSELF.(CURRPATH)`)
          this.defineDfp()
        }
      }
    }
  }
</script>
<style scoped>
  .ad-container { margin-top: 20px; }
  .ad-container.margin-top-30px { margin-top: 30px; }
  .ad-container.margin-top-0 { margin-top: 0; }
  .ad-container.center { display: flex; justify-content: center; margin-right: auto; margin-left: auto; }  
</style>
