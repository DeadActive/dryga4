<template>
  <div id="app">
    <div id="map">
      <div class="custom-control">
        <select name="drone" id="drone" v-model="drone">
          <option
            v-for="(drone, index) in droneModels"
            :key="index"
            :value="drone"
            >{{ drone.name }}</option
          >
        </select>
        <div class="input-wrapper">
          <label for="height">Высота(м) :</label>
          <input type="number" v-model="height" name="height" :step="1" />
        </div>
        <div class="input-wrapper">
          <label for="px">Px% : </label>
          <input type="number" v-model="px" name="px" :step="1" />
        </div>
        <div class="input-wrapper">
          <label for="py">Py% : </label>
          <input type="number" v-model="py" name="py" :step="1" />
        </div>
        <div class="input-wrapper">
          <label for="speed">Скорость(м/с) : </label>
          <input type="number" v-model="speed" name="speed" :step="1" />
        </div>
        <div class="input-wrapper">
          <label for="offset">Оффсет : </label>
          <input
            type="number"
            v-model="offset"
            name="offset"
            :step="0.01"
            :min="1.0"
          />
        </div>
        <div v-if="drone" class="info-wrapper">
          GSD: {{ gsd.toFixed(4) }} m/px <br />
          Время полета: {{ flightTime }} с <br />
          <input
            type="button"
            @click.prevent="createRoutes()"
            value="Построить"
          />
          <input type="button" value="Срезать" @click.prevent="slice()" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import L from 'leaflet'
import {
  coordAll,
  flip,
  bbox,
  square,
  bboxPolygon,
  distance,
  centroid,
  polygon,
  lineIntersect,
  lineSlice,
  point,
  booleanPointInPolygon,
  length,
  polygonToLine,
  booleanPointOnLine,
  nearestPointOnLine,
  transformScale,
  bearing,
  rhumbBearing,
} from '@turf/turf'
import { lineSliceAtIntersection } from 'turf-line-slice-at-intersection'
import 'normalize.css'
import 'leaflet/dist/leaflet.css'
import 'leaflet-path-transform'
import 'leaflet-draw'
import 'leaflet-draw/dist/leaflet.draw.css'
import { isNull } from 'util'

export default {
  name: 'app',
  data() {
    return {
      center: L.latLng(55.14454538318123, 37.456169128417976),
      url: 'http://{s}.tile.osm.org/{z}/{x}/{y}.png',
      map: null,
      params: null,
      drone: null,
      features: null,
      polygon: null,
      height: 80,
      px: 70,
      py: 35,
      drawControl: null,
      bSquare: null,
      route: null,
      angle: 0,
      prevYaw: 0,
      speed: 10,
      flightTime: 0,
      offset: 1.0,
      polygonInit: {
        type: 'Feature',
        properties: {},
        geometry: {
          type: 'Polygon',
          coordinates: [
            [
              [37.45449542999267, 55.1442204061936],
              [37.45447397232056, 55.14200068127899],
              [37.45987057685852, 55.14200068127899],
              [37.45980620384216, 55.1442694594934],
              [37.45449542999267, 55.1442204061936],
            ],
          ],
        },
      },
      droneModels: [
        {
          name: 'Phantom 3 St,Adv,Pro, Phantom 4',
          widthPx: 4000,
          heightPx: 3000,
          width: 6.24,
          height: 4.68,
          focal: 3.61,
          pixelSize: 0.00156,
        },
        {
          name: 'Phantom 4Pro/Adv',
          widthPx: 5472,
          heightPx: 3648,
          width: 12.86,
          height: 8.57,
          focal: 8.57,
          pixelSize: 0.00235,
        },
        {
          name: 'Mavic 2 Pro',
          widthPx: 5472,
          heightPx: 3648,
          width: 13.19,
          height: 8.79,
          focal: 10.26,
          pixelSize: 0.00241,
        },
      ],
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.map = L.map('map').setView(this.center, 16)
      L.tileLayer(this.url, {
        attribution:
          '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors',
      }).addTo(this.map)

      this.features = new L.FeatureGroup()
      this.map.addLayer(this.features)

      this.polygon = L.polygon(coordAll(flip(this.polygonInit))).addTo(
        this.features
      )

      this.bSquare = L.geoJSON(
        bboxPolygon(this.squreBbox(bbox(this.polygon.toGeoJSON())))
      ).addTo(this.features)

      this.route = L.polyline(
        [
          [0, 0],
          [1, 1],
        ],
        { transform: true }
      ).addTo(this.features)

      this.drawControl = new L.Control.Draw({
        position: 'topright',
        draw: false,
        edit: {
          featureGroup: this.features,
        },
      })
      this.map.addControl(this.drawControl)

      this.initEvents()
    })
  },
  methods: {
    slice: function() {
      let route = flip(this.route.toGeoJSON())
      let poly = flip(
        transformScale(this.polygon.toGeoJSON(), parseFloat(this.offset))
      )
      let polyL = transformScale(poly, parseFloat(this.offset) + 0.01)
      let insersections = lineIntersect(route, poly)
      let intersectionPoints = insersections.features.map(d => {
        return d.geometry.coordinates
      })
      let result = []
      for (let i = 0; i < intersectionPoints.length - 1; i++) {
        result.push(
          lineSlice(
            point(intersectionPoints[i]),
            point(intersectionPoints[i + 1]),
            route
          )
        )
      }

      let inside = []
      for (let i = 0; i < result.length; i++) {
        let start = point(result[i].geometry.coordinates[0])
        let end = point(
          result[i].geometry.coordinates[
            result[i].geometry.coordinates.length - 1
          ]
        )
        let mid = result[i].geometry.coordinates[2]

        if (
          booleanPointInPolygon(end, polyL) &&
          booleanPointInPolygon(start, polyL) &&
          !mid
        ) {
          inside.push(result[i])
        }
      }

      //debugger

      let final = []
      let lastBearing = 0
      let reverseBearing = false
      for (let i = 0; i < inside.length; i++) {
        let c = coordAll(inside[i])
        let n = []

        if (i == 0) {
          lastBearing = bearing(c[0], c[1])
        }
        if (Math.abs(bearing(c[0], c[1]) - lastBearing) > 0.01) {
          reverseBearing = true
          lastBearing = bearing(c[0], c[1])
        }
        if (reverseBearing) {
          if (i % 2 == 0) {
            final.push(c[0], c[1])
          } else {
            final.push(c[1], c[0])
          }
          lastBearing = bearing(c[0], c[1])
        } else {
          if (i % 2 != 0) {
            final.push(c[0], c[1])
          } else {
            final.push(c[1], c[0])
          }
          lastBearing = bearing(c[0], c[1])
        }
      }

      this.route.setLatLngs(final)
      this.route.transform.reset()
      this.route.transform.disable()

      this.flightTime = (
        (length(this.route.toGeoJSON()) * 1000) /
        this.speed
      ).toFixed(0)
    },
    onEditVertex: function(e) {
      let box = bbox(e.poly.toGeoJSON())
      let fixedBox = this.squreBbox(box)
      let bPoly = bboxPolygon(fixedBox)
      this.polygon = e.poly
      this.bSquare.clearLayers()
      this.bSquare.addData(bPoly)
    },
    initEvents: function() {
      this.map.on('draw:editvertex', e => this.onEditVertex(e))
      this.route.addEventListener('rotateend', function(e) {
        this.route = e.layer
      })
    },
    squreBbox: function(box) {
      let west = box[0]
      let south = box[1]
      let east = box[2]
      let north = box[3]

      let horizontal = distance(box.slice(0, 2), [east, south])
      let vertical = distance(box.slice(0, 2), [west, north])

      if (horizontal >= vertical) {
        let vertMidpoint = (south + north) / 2
        let horMidpoint = (west + east) / 2
        return [
          horMidpoint + (east - west),
          vertMidpoint - (east - west) / 2,
          horMidpoint - (east - west),
          vertMidpoint + (east - west) / 2,
        ]
      } else {
        let horMidpoint = (west + east) / 2
        let vertMidpoint = (south + north) / 2
        return [
          horMidpoint + (north - south) * 2,
          vertMidpoint - (north - south),
          horMidpoint - (north - south) * 2,
          vertMidpoint + (north - south),
        ]
      }
    },
    rotatePoints: function(center, points, yaw, prevYaw) {
      var res = []
      var angle = (yaw - prevYaw) * (Math.PI / 180)
      for (var i = 0; i < points.length; i++) {
        var p = points[i]
        // translate to center
        // rotate using matrix rotation
        var p3 = [
          center[0] +
            Math.cos(angle) * (p[0] - center[0]) -
            Math.sin(angle) * (p[1] - center[1]),
          center[1] +
            Math.sin(angle) * (p[0] - center[0]) +
            Math.cos(angle) * (p[1] - center[1]),
        ]
        // translate back to center
        // done with that point
        res.push(p3)
      }
      return res
    },
    projectPoints: function(points) {
      let res = []

      for (var i = 0; i < points.length; i++) {
        let p = L.CRS.EPSG3857.latLngToPoint(
          L.latLng(points[i]),
          this.map.getZoom()
        )
        res.push([p.x, p.y])
      }

      return res
    },
    unprojectPoints: function(points) {
      let res = []

      for (var i = 0; i < points.length; i++) {
        let p = L.CRS.EPSG3857.pointToLatLng(
          L.point(points[i]),
          this.map.getZoom()
        )
        res.push([p.lat, p.lng])
      }

      return res
    },
    changeAngle: function() {
      let bSquareGeoJSON = this.bSquare.toGeoJSON()
      let center = this.projectPoints([
        centroid(bSquareGeoJSON).geometry.coordinates,
      ])
      let projectedPoints = this.projectPoints(coordAll(bSquareGeoJSON))
      let rotatedPoints = this.rotatePoints(
        center[0],
        projectedPoints,
        this.angle,
        this.prevYaw
      )
      let unprojectedPoints = this.unprojectPoints(rotatedPoints)
      this.prevYaw = this.angle
      let newSquare = polygon([unprojectedPoints])
      this.bSquare.clearLayers()
      this.bSquare.addData(newSquare)
    },
    getDx: function() {
      let tl = coordAll(flip(this.bSquare.toGeoJSON()))[1]
      let br = coordAll(flip(this.bSquare.toGeoJSON()))[3]
      return L.latLng(tl).distanceTo(L.latLng(tl[0], br[1]))
    },
    getDy: function() {
      let tl = coordAll(flip(this.bSquare.toGeoJSON()))[1]
      let br = coordAll(flip(this.bSquare.toGeoJSON()))[3]
      return L.latLng(tl).distanceTo(L.latLng(br[0], tl[1]))
    },
    getShotsCount: function() {
      return Math.ceil(this.getDx() / this.bx + 3)
    },
    getRoutesCount: function() {
      return Math.ceil(this.getDy() / this.by + 1)
    },
    rectSize: function() {
      let tl = coordAll(flip(this.bSquare.toGeoJSON()))[1]
      let br = coordAll(flip(this.bSquare.toGeoJSON()))[3]

      let wh = {
        width: L.latLng(tl).distanceTo(L.latLng(tl[0], br[1])),
        height: L.latLng(tl).distanceTo(L.latLng(br[0], tl[1])),
      }
      return wh
    },
    createRoutes: function() {
      let routeLatlngs = []
      let size = this.rectSize()
      let tl = coordAll(flip(this.bSquare.toGeoJSON()))[2]
      let R = 6378137
      let dLat = this.by / R

      for (let i = 0; i < this.getRoutesCount(); i++) {
        let dLon =
          this.bx /
          (R * Math.cos((Math.PI * (tl[0] - (dLat * i * 180) / Math.PI)) / 180))
        if (i % 2 == 0) {
          routeLatlngs.push([tl[0] - (dLat * i * 180) / Math.PI, tl[1]])
          routeLatlngs.push([
            tl[0] - (dLat * i * 180) / Math.PI,
            tl[1] + (dLon * (this.getShotsCount() - 3) * 180) / Math.PI,
          ])
        } else {
          routeLatlngs.push([
            tl[0] - (dLat * i * 180) / Math.PI,
            tl[1] + (dLon * (this.getShotsCount() - 3) * 180) / Math.PI,
          ])
          routeLatlngs.push([tl[0] - (dLat * i * 180) / Math.PI, tl[1]])
        }

        //for (let j = 0; j < this.getShotsCount(); j++) {
        //  let n = i % 2 == 0 ? j : this.getShotsCount() - j - 1

        //  routeLatlngs.push([
        //    tl[0] - (dLat * (i - 0.5) * 180) / Math.PI,
        //    tl[1] + (dLon * (n - 1.5) * 180) / Math.PI,
        //  ])
        //}
      }
      this.route.setLatLngs(routeLatlngs)
      this.route.transform.reset()
      this.route.transform.disable()
      this.route.transform.enable({ rotation: true, scaling: false })
    },
  },
  computed: {
    gsd: function() {
      let gsdH =
        (this.height * (this.drone.height / 1000)) /
        ((this.drone.focal / 1000) * this.drone.heightPx)
      let gsdW =
        (this.height * (this.drone.width / 1000)) /
        ((this.drone.focal / 1000) * this.drone.widthPx)

      return gsdH > gsdW ? gsdH : gsdW
    },
    bx: function() {
      return ((this.drone.widthPx * (100 - this.px)) / 100) * this.gsd
    },
    by: function() {
      return ((this.drone.heightPx * (100 - this.py)) / 100) * this.gsd
    },
    dx: function() {
      let tl = coordAll(flip(this.bSquare.toGeoJSON()))[1]
      let br = coordAll(flip(this.bSquare.toGeoJSON()))[3]
      return L.latLng(tl).distanceTo(L.latLng(tl[0], br[1]))
    },
    dy: function() {
      let tl = coordAll(flip(this.bSquare.toGeoJSON()))[1]
      let br = coordAll(flip(this.bSquare.toGeoJSON()))[3]
      return L.latLng(tl).distanceTo(L.latLng(br[0], tl[1]))
    },
    shotsCount: function() {
      return Math.ceil(this.dx / this.bx + 3)
    },
    routesCount: function() {
      return Math.ceil(this.dy / this.by + 1)
    },
  },
  components: {},
}
</script>

<style lang="scss">
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

#map {
  position: absolute;
  width: 100%;
  height: 100vh;
  display: flex;
  justify-content: flex-start;
  align-content: center;
  flex-direction: row;
}

.custom-control {
  display: flex;
  justify-content: space-between;
  align-content: center;
  flex-direction: column;
  margin-top: 100px;
  margin-left: 10px;
  max-height: 30vh;
  padding: 10px;
  background: #fff;
  border-radius: 5px;
  color: #000;
  z-index: 999;
  -webkit-box-shadow: 5px 5px 10px 1px rgba(173, 173, 173, 1);
  -moz-box-shadow: 5px 5px 10px 1px rgba(173, 173, 173, 1);
  box-shadow: 5px 5px 10px 1px rgba(173, 173, 173, 1);
}
</style>
