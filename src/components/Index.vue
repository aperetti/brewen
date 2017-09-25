<template>
  <q-layout ref="layout" view="hHr LpR lFf" :right-breakpoint="1100">
    <div slot="left">
      <q-list no-border link >
        <q-list-header>Navigation</q-list-header>
        <q-side-link item to="/login">
          <q-item-side icon="exit_to_app" /> 
          <q-item-main label="Logout" />
        </q-side-link>
        <q-side-link item icon="assignment" to="/brew">
          <q-item-side icon="assignment" />
          <q-item-main label="Brew" />
        </q-side-link>
        <q-side-link item to="/monitor">
          <q-item-side icon="trending_up" />
          <q-item-main label="Monitor" />
        </q-side-link>
      </q-list>
    </div>
    <q-toolbar slot="header" class="toolbar">
      <q-btn class="hide-on-drawer-visible" @click="$refs.leftDrawer.open()">
        <q-icon name="menu" />
      </q-btn>
      <q-toolbar-title>
       Brewen 
      </q-toolbar-title>
    </q-toolbar>
    <router-view :brewProp="brewId" v-on:brew="setBrew" ref="router" class="layout-view container"></router-view>
  </q-layout>
</template>

<script>
import { QLayout, QList, QListHeader, QSideLink, QItemSide, QItemMain, QToolbar, QToolbarTitle, QIcon, QBtn } from 'quasar-framework'
import mqtt from 'mqtt'
import _ from 'lodash'
import Brew from './Index/Brew'
export default {
  components: {
    Brew,
    QLayout,
    QList,
    QListHeader,
    QSideLink,
    QItemSide,
    QItemMain,
    QToolbar,
    QToolbarTitle,
    QIcon,
    QBtn
  },
  data () {
    return {
      brewId: null
    }
  },
  mounted () {
    var client = mqtt.connect('wss://brew.photoredux.com:9095')
    client.on('connect', () => {
      console.log('loaded')
      client.subscribe('/sensor/#', {qos: 1}, function (topic) {
        console.log('Subcribed to temperature reading ', topic)
      })
      client.subscribe('/relay/#', {qos: 1}, (topic) => console.log('Subscribed to relay ', topic))
      client.subscribe('/brew', {qos: 1}, (topic) => console.log('Subscribed to brew notification ', topic))
      client.on('message', (topic, message) => {
        var topics = topic.split('/')
        if (topics[1] === 'sensor' && topics[2] === 'ds18b20') {
          if (topics.length === 4) {
            let sensor = topics[3]
            if (this.$refs.router) {
              this.$refs.router.$emit('sensor', sensor, _.toNumber(message.toString()))
            }
          }
        }
        else if (topics[1] === 'relay' && topics[2] === 'status') {
          this.$refs.router.$emit('relay', !_.toNumber(message.toString()))
        }
        else if (topics[1] === 'brew') {
          this.$refs.router.$emit('brewUpdate', message.toString())
        }
      })
    })
  },
  computed: {
    position () {
      const transform = `rotateX(${this.rotateX}deg) rotateY(${this.rotateY}deg)`
      return {
        top: this.moveY + 'px',
        left: this.moveX + 'px',
        '-webkit-transform': transform,
        '-ms-transform': transform,
        transform
      }
    }
  },
  methods: {
    setBrew (id) {
      this.brewId = id
    }
  }
}
</script>

<style lang="stylus">
.container
  width 100%
  max-width 1200px
  display inline-block
  position relative;

.logo-container
  width 192px
  height 268px
  perspective 800px
  position absolute
  top 50%
  left 50%
  transform translateX(-50%) translateY(-50%)

.logo
  position absolute
  transform-style preserve-3d

.layout-view
  padding 10px
</style>
