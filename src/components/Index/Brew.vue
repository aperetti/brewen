<template>
  <div>
    <div class="row shadow-2 full-width" >
        <q-toolbar color="primary">
            <q-select class="col-6" type="radio" v-model="brewId" :options="brewList" dark label="Select Brew"></q-select>
            <q-btn flat @click="deleteBrew"> <q-icon name="delete" /></q-btn>
            <q-btn flat @click="newBrew">
              <q-icon name="add" /> New Brew
            </q-btn>
            <q-btn flat @click="edit = !edit"> <q-icon :name="edit? 'format_clear': 'mode_edit'" /></q-btn>
        </q-toolbar>
      <div class="row sm-gutter" style="padding:10px;">
        <div class="full-width row sm-gutter">
          <div v-if="brew.name" class="col-12">
            <div class="row wrap sm-gutter">
              <div class="col-sm-3 col-xs-12">
                <q-input stack-label="Brew" :readonly=!edit class="full-width" v-model="brew.name" />
              </div>
              <div class="col-sm-1 col-xs-6">
                <q-input stack-label="Batch Size" class="full-width" :readonly=!edit v-model.number="brew.batchSize" />
              </div>
              <div class="col-sm-2 col-xs-6">
                <q-select type="radio" stack-label="Status" v-model="brew.status" class="full-width" :options="statusEnum" label="Status"></q-select>
              </div>
              <div class="col-sm-2">
                <q-select type="radio" stack-label="Sensor" v-model="brew.sensorId" class="full-width" :options="sensorList" label="Sensor"></q-select>
              </div>
              <div class="col-sm-4">
                <q-datetime :readonly=!edit v-model="brew.date" stack-label="Brew Date" type="date" class="full-width" label="Brew Date"></q-datetime>
              </div>
            </div>
            <div>
              <q-input type="textarea" stack-label="Notes" :min-rows="2" class="full-width" v-model="brew.notes"></q-input>
            </div>
          </div>
        </div>
        <div class="col-12" v-if="brew && brew.steps">
          <div class="row xs-gutter">
            <div class="col-12">
            <q-pagination v-model="step" :max="brew.steps.length"></q-pagination>
            </div>
            <div class="col-12">
            <h2>{{brew.steps[step-1].name}}</h2>
            </div>
            <div class="col-12">
            <blockquote>{{brew.steps[step-1].recommendations}}</blockquote>
            </div>
            <div class="col-12">
            <q-input type="textarea" stack-label="Notes" :min-rows="10" class="full-width" v-model="brew.steps[step-1].notes"></q-input>
            </div>
            <div class="col-12">
            <div class="row xs-gutter" style="padding-top:10px">
              <div class="col-3 xs-col-4">
                <q-input stack-label="Target Temperature (°F)" class="full-width" :readonly=!edit v-model.number="brew.steps[step-1].targetTemp" />
              </div>
              <div class="col-3 xs-col-4">
                <q-select stack-label="Alarm Mode" :readonly=!edit :options='alarmModes' v-model='alarmMode' label='Temperature Alarm'></q-select>
              </div> 
              <div class="col-3 xs-col-4">
                <q-input stack-label="Timer (minutes)" class="full-width" :readonly="!edit" v-model="brew.steps[step-1].timer" />
              </div> 
            </div>
            </div>
            <div class="row full-width">
              <div class="col-6" v-if="sensorTemp">
                <h4>{{sensorTemp}}°F</h4>
              </div>
              <div>
              </div>
            </div>
            <div class="row full-width">
              <div class="col-12" v-if="brew.steps[step-1].timer">
                <label>Time (minutes)</label>
                <q-progress  :percentage="taskPercent" :class="['dark', brew.steps[step-1].endTime === 0 ? 'stripe animate' : '']"></q-progress>
              </div>
            </div>
            <div class="col-12">
            <q-btn @click="start(step-1)" v-if="brew.steps[step-1].status === 'Planning'">Start</q-btn>
            <q-btn v-if="brew.steps[step-1].status === 'In Progress'" @click="finish(step-1)">Finish</q-btn>
            <q-btn v-if="brew.steps[step-1].status === 'Complete'" @click="restart(step-1)">Restart</q-btn> 
          </div>  
          </div>
        </div>
      </div>
    </div>  
  </div>
</template>
<script>
  const modes = [
    {label: 'No Alarm', value: null},
    {label: 'Maintain (Min)', value: 'min'},
    {label: 'Maintain (Max)', value: 'max'},
    {label: 'Target', value: 'target'}
  ]

  function showNotification () {
  }
  var timeout
  import { QProgress, QInput, QPagination, QDatetime, Toast, QToggle, QSelect, QBtn, QIcon, QToolbar } from 'quasar-framework'
  import Brew from '../../api/brew'
  import monitor from '../../api/monitor'
  export default {
    name: 'brew',
    components: {
      QProgress,
      QInput,
      QPagination,
      QDatetime,
      Toast,
      QToolbar,
      QToggle,
      QSelect,
      QBtn,
      QIcon
    },
    data () {
      return {
        alarm: false,
        alarmMode: null,
        alarmModes: modes,
        sensorTemp: null,
        taskPercent: 0,
        statusEnum: [],
        brewId: 0,
        edit: true,
        brew: {},
        brewList: [],
        step: 1,
        sensorList: [],
        disableWatch: false,
        changed: false
      }
    },
    props: ['brewProp'],
    mounted () {
      showNotification()
      monitor.getSensorBuffer(true).then(res => {
        this.sensorList = res.data.map(el => Object.assign({}, {label: el.alias || el.sensor, value: el.sensor}))
      })
      this.$on('brewUpdate', (id) => {
        if (id === this.brewProp) {
          this.syncBrew()
          this.syncBrewList()
          Toast.create({
            html: 'Brew synced!',
            timeout: 1500,
            icon: 'info',
            button: {
              color: '#fff'
            }
          })
        }
      })
      this.$on('sensor', (id, temp) => {
        if (id === this.brew.sensorId) {
          this.sensorTemp = Math.round(10 * temp) / 10
        }
      })
      this.brewId = this.brewProp
      setInterval(() => {
        this.calcPercent()
      }, 1000)
      this.syncBrewList()
      Brew.getStatusEnum().then(res => {
        this.statusEnum = res.data.map(el => Object.assign({}, {label: el, value: el}))
      })
      if (this.brewId) {
        this.syncBrew()
      }
    },
    computed: {
    },
    watch: {
      brew: {
        handler: function (val, oldVal) {
          if (!this.disableWatch) {
            clearTimeout(timeout)
            timeout = setTimeout(() => {
              console.log('test')
              this.saveBrew()
              this.syncBrewList()
            }, 1000)
          }
        },
        deep: true
      },
      brewId: function (val, oldVal) {
        if (!this.disableWatch) {
          console.log('Syncing brew')
          this.syncBrew()
          this.$emit('brew', this.brewId)
        }
      }
    },
    methods: {
      calcPercent () {
        // Gets the current step
        var step = this.brew && this.brew.steps ? this.brew.steps[this.step - 1] : {}
        if (step.endTime) {
          this.taskPercent = 100
        }
        else if (step.startTime) {
          var percent = 100 * ((Date.now() - step.startTime) / (1000 * 60)) / step.timer
          this.taskPercent = percent
          if (percent > 100) {
            console.log('finished')
          }
        }
        else {
          this.taskPercent = 0
        }
      },
      timeElapsed (st) {
        var step = this.brew.steps[st]
        if (step.endTime) {
          return (step.endTime - step.startTime) / 1000 / 60
        }
        else if (step.startTime) {
          return (step.startTime - Date.now()) / 1000 / 60
        }
        else {
          return 0
        }
      },
      start (step) {
        this.brew.steps[step].status = 'In Progress'
        this.brew.steps[step].startTime = Date.now()
      },
      finish (step) {
        this.brew.steps[step].status = 'Complete'
        this.brew.steps[step].endTime = Date.now()
      },
      restart (step) {
        this.brew.steps[step].status = 'Planning'
        this.brew.steps[step].endTime = 0
        this.brew.steps[step].startTime = 0
      },
      syncBrewList () {
        Brew.getList().then((res) => {
          this.brewList = res.data.map(el => Object.assign({}, {label: el.name, value: el.id}))
        })
      },
      syncBrew () {
        if (this.brewId) {
          this.disableWatch = true
          Brew.getById(this.brewId).then(res => {
            this.brew = Object.assign({}, res.data)
            this.syncBrewList()
            this.$nextTick(() => {
              this.disableWatch = false
            })
          }).catch(e => {
            this.disableWatch = false
            Toast.create.negative({
              html: 'Could not grab Brew',
              timeout: 2500,
              button: {
                color: '#fff'
              }
            })
          })
        }
      },
      saveBrew () {
        Brew.postById(this.brew.id, this.brew).then((res) => {
        })
      },
      newBrew () {
        Brew.postNew().then((res) => {
          this.brew = res.data
          this.brewId = res.data.id
        })
      },
      deleteBrew () {
        Brew.deleteById(this.brew.id).then(res => {
          this.brew = []
          this.syncBrewList()
        })
      }
    }
  }
</script>
