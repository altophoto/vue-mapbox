<template></template>

<script>

  import mixin from './layerMixin';
  import layerEvents from '../../lib/layerEvents';

  export default {
    mixins: [mixin],
    props: {
      coordinates: {
        type: Array,
        required: true
      },
      url: {
        type: String,
        required: true
      }
    },
    data() {
      return {
        source: undefined
      }
    },

    watch: {
      coordinates(val) {
        if (this.initial) return
        this.source.setCoordinates(val)
      }
    },

    methods: {
      _deferredMount(payload) {
        const source = {
          type: 'image',
          url: this.url,
          coordinates: this.coordinates
        }

        this.map = payload.map;
        this.map.on('dataloading', this._watchSourceLoading)
        try {
          this.map.addSource(this.sourceId, source)
        } catch (err) {
          if (this.replaceSource) {
            this.map.removeSource(this.sourceId);
            this.map.addSource(this.sourceId, source)
          } else {
            this._emitMapEvent('mgl-layer-source-error', { sourceId: this.sourceId, error: err })
          }
        }
        this.source = this.map.getSource(this.sourceId)
        this._addLayer()
        if (this.listenUserEvents) {
          this._bindEvents(layerEvents);
        }
        this.initial = false;
        payload.component.$off('mgl-load', this._deferredMount)
      },

      _addLayer() {
        let existed = this.map.getLayer(this.layerId);
        if (existed) {
          if (this.replace) {
            this.map.removeLayer(this.layerId);
          } else {
            this._emitMapEvent('mgl-layer-exists', { layerId: this.layerId });
            return existed;
          }
        }
        let layer = {
          id: this.layerId,
          source: this.sourceId,
          type: 'raster'
        }
        if (this.refLayer) {
          layer.ref = this.refLayer;
        } else {
          if (this['source-layer']) {
            layer['source-layer'] = this['source-layer']
          }
          if (this.minzoom) layer.minzoom = this.minzoom
          if (this.maxzoom) layer.maxzoom = this.maxzoom
          if (this.layout) {
            layer.layout = this.layout;
          }
          if (this.filter) layer.filter = this.filter
        }
        layer.paint = this.paint ? this.paint : { 'raster-opacity': 1 };
        layer.metadata = this.metadata

        this.map.addLayer(layer, this.before);
        this._emitMapEvent('mgl-layer-added', { layerId: this.layerId });
      }
    }
  }
</script>
