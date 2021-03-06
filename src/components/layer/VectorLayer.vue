<template></template>

<script>

import layerEvents from '../../lib/layerEvents';
import mixin from './layerMixin';

export default {
  mixins: [mixin],
  props: {
    url: {
      type: String,
      default: undefined
    },
    tiles: {
      type: Array,
      default: () => []
    },
    tilesMinZoom: {
      type: Number,
      default: undefined
    },
    tilesMaxZoom: {
      type: Number,
      default: undefined
    },
    filter: {
      type: Array,
      default:undefined
    }
  },

  watch: {
    filter(filter) {
      if (this.initial) return;
      this.map.setFilter(this.layerId, filter);
    }
  },

  methods: {
    _deferredMount(payload) {
      if (payload.mapId !== this.mapId) return;
      this.map = payload.map;
      let source = {
        type: 'vector',
        tilesMinZoom: this.tilesMinZoom,
        tilesMaxZoom: this.tilesMaxZoom,
        url: this.url,
        tiles: this.tiles
      }

      this.map.on('dataloading', this._watchSourceLoading);
      try {
        this.map.addSource(this.sourceId, source)
      } catch (err) {
        if (this.replaceSource) {
          this.map.removeSource(this.sourceId);
          this.map.addSource(this.sourceId, source)
        } else {
          this._emitMapEvent('mgl-layer-source-error', { sourceId: this.sourceId, error: err });
        }
      }
      this._addLayer()
      if (this.listenUserEvents) {
        this._bindEvents(layerEvents)
      }
      this.map.off('dataloading', this._watchSourceLoading)
      this.initial = false
      mapComponent.$on('mgl-load', this._deferredMount)
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
        source: this.sourceId
      }
      if (this.refLayer) {
        layer.ref = this.refLayer;
      } else {
        layer.type = this.type ? this.type : 'fill'
        layer.source = this.sourceId;
        layer['source-layer'] = this['source-layer']
        if (this.minzoom) layer.minzoom = this.minzoom
        if (this.maxzoom) layer.maxzoom = this.maxzoom
        if (this.layout) {
          layer.layout = this.layout;
        }
        if (this.filter) layer.filter = this.filter
      }
      layer.paint = this.paint
                          ? this.paint
                          : {'fill-color': `rgba(${12 * (this.layerId.length * 3)},153,80,0.55)` };
      layer.metadata = this.metadata

      this.map.addLayer(layer, this.before);
      this._emitMapEvent('mgl-layer-added', { layerId: this.layerId });
    }
  }
}
</script>
