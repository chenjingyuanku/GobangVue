<template>
  <div id="app">
    <transition :name="transitionName">
      <router-view/>>
    </transition>
  </div>
</template>

<script lang="ts">
import {Route} from 'vue-router';
import { Component, Watch, Vue } from 'vue-property-decorator';
@Component
export default class App extends Vue{
  transitionName:string = "slide-left";
  @Watch('$route')
  onChildChanged(val:Route, oldVal:Route) {
      if(val.name === "home"){
        this.transitionName = "slide-right";
      }else{
        this.transitionName = "slide-left";
      }
  }
}
</script>


<style>

  /*路由切换动画*/
  /*
  */
  .slide-right-enter-active,
  .slide-left-enter-active{
    will-change: transform;
    transition: all 0.6s ease-out;
  }
  .slide-right-leave-active,
  .slide-left-leave-active {
    will-change: transform;
    transition: all 0.6s ease-out;
    position: absolute;
    left: 0%;
    right: 0%;
  }
  .slide-right-enter {
  opacity: 0;
    transform: translate(-100%, 0);
  }
  .slide-right-leave-active {
  opacity: 0;
    transform: translate(100%, 0);
  }
  .slide-left-enter {
  opacity: 1;
    transform: translate(100%, 0);
  }
  .slide-left-leave-active {
  opacity: 0;
    transform: translate(-100%, 0);
  }

</style>


