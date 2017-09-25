<template>
  <div>
    <div class="before"></div>
    <div class="row">
    <div class="col-12" style="margin-bottom: 10px;">
      <div class="" style="padding: 0px; margin: 0 auto">
          <div ref="chartLayout" >
            <canvas class="chart div div-2dp" id="myChart" ref="myCanvas"></canvas>
          </div>
      </div>
    </div>
    <q-list style="padding: 5px; margin: 0 auto" class="col-12">
      <q-collapsible opened icon="assessment" label="Temperature Sensors">
          <q-list sparse no-border highlight> 
            <q-item :key='"temps"-index' v-for="(sen, index) in sortedSensors">
              <q-item-main >
                <q-item-tile label>
                  <b>{{sen.alias || sen.sensor}}</b>
                </q-item-tile>
                <q-item-tile sublabel>
                    <span>
                    {{Math.round(sen.temp*100)/100}}­°F
                    </span>
                </q-item-tile>
              </q-item-main>
              <q-item-side right>
                <q-item-tile icon="alarm" />
                <q-item-tile>
                  {{sen.targetTemp ? sen.targetTemp : "--"}}°F
                </q-item-tile>
              </q-item-side>  
            </q-item>
          </q-list>
        <q-btn class="outline primary small" @click="$refs.sensorAliases.open()"><q-icon name="edit"/> Aliases</q-btn>
      </q-collapsible>
      <q-collapsible icon="whatshot" label="Word Chill Calculator">
              <q-select float-label="Wort Sensor" list name="wortSensor" v-model="wortSensor" :options="availableSensors">
              </q-select>
              <q-input float-label="Target Temp" class="full-width" v-model="targetTemp" />
            <q-select :disabled="overrideAmbient" float-label="Ambient Sensor" name="ambient-sensor" v-model="ambientSensor" :options="availableSensors">
            </q-select>
              <q-input float-label="Ambient Temp" :disable="!overrideAmbient" type="number" v-model="ambientTemp"></q-input>
            <label>
              <q-toggle v-model="overrideAmbient" id="my-test1" name="my-test1" class="md-primary"></q-toggle>
              Override Ambient Sensor
            </label>
            {{Math.round(chillTime / 1000 / 60)}} Minutes
            <q-tooltip anchor="bottom middle" self="top middle">Chill Rate: {{k.toExponential()}}</q-tooltip>
      </q-collapsible>
      <q-collapsible icon="power" label="Power Outlet">
        <div style="margin-bottom: 10px;">
          <div style="width: 100%" md-elevation='5' >
            <div md-tag="md-toolbar" class="md-dense" md-elevation="4">
              <div :class="'md-headline'">Relay</div>
            </div>
              <div md-flex="100">
                <q-btn @click="toggleRelay" style='margin: 0px 0px 0px 0px; width:100%; height: 100%' :class="['light', relayStatus ? 'red' : '']">Toggle</q-btn>
              </div>
          </div>
        </div>
      </q-collapsible>
    </q-list>
  </div>
    <q-modal ref="sensorAliases" :content-css="{padding: '20px'}">
      <h4>Sensor Aliases</h4>
      <div class="row xs-gutter" v-for="(sensor, index) in sensorData">
          <div class="col-6">
              <q-input float-label="Sensor Serial"  v-model="sensorData[index].sensor" />
          </div>
          <div class="col-6">
              <q-input float-label="Sensor Alias" v-model="sensorData[index].alias" />
          </div>
      </div>
      <q-btn class="outline primary" @click="$refs.sensorAliases.close()">Cancel</q-btn>
      <q-btn class="primary" @click="saveAliases">Save</q-btn>
    </q-modal>
    <q-modal ref="clearSensorData" :content-css="{padding: '20px'}">
      <div>Are you sure you would like to clear all existing sensor data?</div>
      <q-btn class="outline primary" @click="$refs.clearSensorData.close()">Cancel</q-btn>
      <q-btn class="primary" @click="clearSensors">Clear</q-btn>
    </q-modal>
    <q-modal class="maximized" ref="brew">

    </q-modal>
    <div class="after"></div>
  </div>
</template>

<script>
import Chart from 'chart.js'
import monitor from '../../api/monitor'
import user from '../../api/user'
import _ from 'lodash'

var templateColors = (index, alpha) => {
  let colors = [
    `rgba(255, 99, 132, ${alpha})`,
    `rgba(54, 162, 235, ${alpha})`,
    `rgba(255, 206, 86, ${alpha})`,
    `rgba(75, 192, 192, ${alpha})`
  ]
  return colors[index]
}
var templateDataSet = (i) => {
  return {
    sensor: -1,
    radius: 0.5,
    lineTension: 0.1,
    type: 'line',
    label: `Sensor ${i}`,
    borderWidth: 1,
    fill: true,
    backgroundColor: templateColors(i, 0.2),
    borderColor: templateColors(i, 1),
    data: []
  }
}
function resizeChart () {
  var canvas = document.getElementById('myChart')
  var newCanvasWidth = Math.max((canvas.parentNode.clientWidth), 1200)
  var newCanvasHeight = newCanvasWidth * 0.6
  if ((newCanvasWidth !== canvas.width) || (newCanvasHeight !== canvas.height)) {
    canvas.width = newCanvasWidth
    canvas.height = newCanvasHeight
  }
}

function generateChart () {
  var ctx = window.document.getElementById('myChart')
  return new Chart(ctx, {
    responsive: true,
    maintainAspectRatio: false,
    options: {
      title: {
        display: true,
        text: 'Sensor Data',
        fontSize: 16
      },
      scales: {
        xAxes: [{
          type: 'time',
          scaleLabel: {
            display: true,
            labelString: 'Time (PST)'
          },
          ticks: {
            minRotation: 45
          },
          time: {
            unit: 'minute'
          }
        }],
        yAxes: [{
          scaleLabel: {
            display: true,
            labelString: 'Temperature (°F)'
          },
          display: true,
          ticks: {
            suggestedMin: 0    // minimum will be 0, unless there is a lower value.
          }
        }]
      },
      fullWidth: true,
      maintainAspectRation: false,
      responsive: false
    },
    data: {
      datasets: [
        templateDataSet(1),
        templateDataSet(2),
        templateDataSet(3),
        templateDataSet(4)
      ]
    }
  })
}
const updateInterval = 30 * 1000
var tempChart
import {QBtn, QModal, QCollapsible, QToggle, QTooltip, QSelect, QInput, QItem, QList, QItemTile, QItemSide, QItemMain, QIcon} from 'quasar-framework'
export default {
  name: 'monitor',
  components: {
    QItemSide,
    QItemMain,
    QIcon,
    QModal,
    QItem,
    QItemTile,
    QList,
    QInput,
    QCollapsible,
    QToggle,
    QTooltip,
    QSelect,
    QBtn
  },
  mounted () {
    resizeChart()
    tempChart = generateChart()
    this.fetchSensorData()
    window.setInterval(this.updateGraph, updateInterval)
    this.$on('sensor', (sensor, temp) => {
      this.updateSensor(sensor, temp)
    })
    this.$on('relay', (status) => {
      this.relayStatus = status
    })
    // var client = mqtt.connect('wss://brew.photoredux.com:9095')
    // client.on('connect', () => {
    //   console.log('loaded')
    //   client.subscribe('/sensor/#', {qos: 1}, function (topic) {
    //     console.log('Subcribed to temperature reading ', topic)
    //   })
    //   client.subscribe('/relay/#', {qos: 1}, (topic) => console.log('Subscribed to relay ', topic))
    //   client.on('message', (topic, message) => {
    //     var topics = topic.split('/')
    //     if (topics[1] === 'sensor' && topics[2] === 'ds18b20') {
    //       if (topics.length === 4) {
    //         let sensor = topics[3]
    //         this.updateSensor(sensor, _.toNumber(message.toString()))
    //       }
    //     }
    //     else if (topics[1] === 'relay' && topics[2] === 'status') {
    //       this.relayStatus = !_.toNumber(message.toString())
    //     }
    //   })
    // })
  },
  data () {
    return {
      test: null,
      ambientSensor: null,
      k: 0,
      overrideAmbient: false,
      wortSensor: null,
      targetTemp: null,
      ambientTemp: null,
      fullScreen: false,
      sensorChart: {},
      sensorData: [],
      relayStatus: false
    }
  },
  methods: {
    fetchSensorData () {
      monitor.getSensorBuffer()
      .then(res => {
        console.log(res.data)
        this.sensorData = res.data
        var chart = tempChart.config.data
        this.sensorData.forEach(s => {
          var idx = _.findIndex(chart.datasets, o => o.sensor === s.sensor)
          if (idx === -1) {
            idx = _.findIndex(chart.datasets, o => o.sensor === -1)
            chart.datasets[idx].sensor = s.sensor
          }
          var tempData = s.runningTemp.map(temp => Object.assign({}, {x: Math.round(temp.time / updateInterval) * updateInterval, y: temp.temp}))
          chart.datasets[idx].data = _.uniqBy(tempData, 'x').map(t => Object.assign(t, {x: new Date(t.x)}))
        })
        tempChart.update()
      })
    },
    updateSensor (sensor, temp) {
      var index = _.findIndex(this.sensorData, (o) => o.sensor === sensor)
      if (index === -1) {
        this.sensorData.push({
          sensor: sensor,
          temp: temp,
          runningTemp: [{temp: temp, time: Date.now()}]
        })
      }
      else {
        var update = this.sensorData[index]
        update.temp = temp
        update.lastUpdate = Date.now()
        if (Date.now() - _.last(update.runningTemp).time > 10000) {
          update.runningTemp.push({temp: temp, time: Date.now()})
          update.runningTemp = _.takeRight(update.runningTemp, 300)
        }
      }
    },
    saveAliases () {
      user.setSensorAliases(this.sensorData.map(sensor => {
        return {sensor: sensor.sensor, alias: sensor.alias}
      })).then((res) => {
        console.log(res)
        this.$refs.sensorAliases.close()
      })
    },
    clearSensors () {
      this.sensors = []
      tempChart = generateChart()
      this.fetchSensorData()
    },
    updateGraph () {
      var datasets = tempChart.config.data.datasets
      this.sensorData.map(sensor => {
        var idx = _.findIndex(datasets, o => o.sensor === sensor.sensor)
        if (idx === -1) {
          idx = _.findIndex(datasets, o => o.sensor === -1)
          datasets[idx].sensor = sensor.sensor
        }
        datasets[idx].data.push({x: new Date(), y: sensor.temp})
        datasets[idx].data = _.takeRight(datasets[idx].data, 150)
      })
      tempChart.update()
    },
    toggleRelay: function () {
      monitor.toggleRelay(7)
        .then((res) => console.log(res))
    }
  },
  computed: {
    availableSensors () {
      return this.sensorData.map(sen => {
        return {label: `${sen.alias || sen.sensor}`, value: sen.sensor}
      })
    },
    sortedSensors () {
      return _.sortBy(this.sensorData, (o) => o.sensor)
    },
    chillTime () {
      if (this.targetTemp && this.wortSensor && ((!this.overrideSensor && this.ambientSensor) || (this.ovverrideSesnor && this.ambientTemp))) {
        var tTemp = _.toNumber(this.targetTemp)
        var cSensor = this.sensorData[_.findIndex(this.sensorData, (o) => o.sensor === this.wortSensor)]
        var aSensor = this.sensorData[_.findIndex(this.sensorData, (o) => o.sensor === this.ambientSensor)]
        var aTemp = this.overrideAmbient ? this.ambientTemp : aSensor.temp
        var tempInitial = cSensor.runningTemp.length >= 3 ? _.nth(cSensor.runningTemp, -3).temp : _.head(cSensor.runningTemp).temp
        var tempNow = _.last(cSensor.runningTemp).temp
        var timeInititial = cSensor.runningTemp.length >= 3 ? _.nth(cSensor.runningTemp, -3).time : _.head(cSensor.runningTemp).time
        var timeDelta = _.last(cSensor.runningTemp).time - timeInititial
        var k = Math.log((tempNow - aTemp) / (tempInitial - aTemp)) / (-1 * timeDelta)
        var time = Math.log((tTemp - aTemp) / (tempInitial - aTemp)) / (-1 * k)
        this.k = k
        return time
      }
      else {
        return null
      }
    }
  }
}
</script>


<style scoped>
canvas {
  margin: 0 auto;
  display: block;
}
h1, h2 {
  font-weight: normal;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
.chart {
  padding: 10px;
  margin-bottom: 20px;
  width: 100% !important;
  max-width: 1200px;
  height: auto !important;
}

.before {
    width: 5%;
    min-width: 10px;
    max-width: 150px;
    display: inline-block;
}
.after {
   content: '1';
    min-width: 10px;
    width: 5%; 
    display: inline-block;
}
</style>
