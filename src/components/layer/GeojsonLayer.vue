<template></template>

<script>
import bus from '../../messageBus';
import layerEvents from '../../lib/layerEvents';
import mixin from './layerMixin';


export default {
  mixins: [mixin],
  props: {
    initSource: {
      type: [Object, String]
    },
    type: {
      validator(value) {
        let allowedValues = ['fill', 'line', 'symbol', 'circle', 'fill-extrusion', 'raster', 'background'];
        return (typeof value === 'string' && allowedValues.indexOf(value) !== -1) || value === undefined;
      },
      default: 'fill'
    },
    initFilter: {
      type: Array,
      default: undefined
    }
  },

  data() {
    return {
      source: this.initSource
    }
  },

  watch: {
    initSource(data) {
      if (this.initial) return;
      this.map.getSource(this.sourceId).setData(data);
      this.source = this.map.getSource(this.sourceId);
    },
    initFilter(filter) {
      if (this.initial) return;
      this.map.setFilter(this.layerId, filter);
      this.mapOptions.filter = filter;
    }
  },

  mounted() {
    this._checkMapId();
    // We wait for "load" event from map component to ensure mapbox is loaded and map created
    bus.$on('mgl-load', this._deferredMount);
  },

  methods: {
    _deferredMount(payload) {
      if (payload.mapId !== this.mapId) return;
      this.map = payload.map;
      this.map.on('dataloading', this._watchSourceLoading);
      if (this.source) {
        try {
          this.map.addSource(this.sourceId, {
            type: 'geojson',
            data: this.source
          })
        } catch (err) {
          if (this.replaceSource) {
            this.map.removeSource(this.sourceId);
            this.map.addSource(this.sourceId, {
              type: 'geojson',
              data: this.source
            })
          } else {
            this._emitMapEvent('mgl-layer-source-error', { sourceId: this.sourceId, error: err });
          }
        }
      }
      this._addLayer();
      if (this.listenUserEvents) {
        this._bindEvents(layerEvents);
      }
      this.map.off('dataloading', this._watchSourceLoading);
      this.initial = false;
      bus.$off('mgl-load', this._deferredMount);
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