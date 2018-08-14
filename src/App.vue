<template>
  <div id="app">
     <center><h1>ACORN SAMPLE</h1></center>
     <button @click='consolelog'>Console Log output</button>
     <button @click='run'>Run</button>
     <textarea v-model='code'></textarea>
     <div class='result'>
      <VueObjectView v-if='see' ref='display' :value="object" />
      </div>
  </div>
</template>

<script>
import VueObjectView from 'vue-object-view';
import acorn from "acorn";
let a = require('acorn');

export default {
  components:{
    "VueObjectView": VueObjectView,
  },
  data() {
    return {
      code: 'let a = 5 * 7',
      object: {},
      see: true,
    }
  },
  methods: {
    consolelog(){
      let node = a.parse(this.code);
      console.log(node);
    },
    run(){
      this.object = a.parse(this.code);
      this.see = false;
      setTimeout( ()=>{
        this.see = true;
      },100 );
    }
  },
  created(){
    this.object = a.parse(this.code);
    console.log(a);
  },
}
</script>

<style>
textarea{
  display: block;
  height: 400px;
  width: 600px;
}
.result{
  margin-top: 100px;
  border-top: 3px solid blue;
}
</style>
