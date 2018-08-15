<template>
  <div id="app">
     <center><h1>ACORN SAMPLE</h1></center>
     <button @click='consolelog'>Console Log output</button>
     <button @click='run'>Run</button>
     <button @click='calculate'>Calculate</button>
     <textarea v-model='code'></textarea>
     <div class='result'>
       <VueObjectView v-if='see' ref='display' :value="object" />
      </div>
      <div class='calculation'>
        <VueObjectView v-if='see2' :value="calculation" />
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
      see2: true,
      calculation: {},
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
    },
    setCalculation(ob){
      this.calculation = ob;
      this.see2 = false;
      setTimeout(()=>{
        this.see2 = true;
      }, 100);
    },
    calculate(){
      // this.setCalculation(this.findAllFunctions());
      // let f = this.pickFuctionByNmae(this.findAllFunctions(), "ispravna");

      // this.setCalculation(this.getFunctionParamsNames(f));
      this.setCalculation(this.findAllVariablesDeclaredOnPage());
    },
    findAllFunctions(){
      let ast = a.parse(this.code);
      let result = [];
      ast.body.forEach(el=>{
        if(el.type==='FunctionDeclaration')
          result.push(el);
      });
      return result;
    },
    pickFuctionByNmae(arrayAstF, name){
      for(let i=0; i<arrayAstF.length; i++)
        if(arrayAstF[i].id.name===name)
          return arrayAstF[i];
      return null;
    },
    getFunctionParamsNames(astF){
      let r = [];
      for(let i=0; i<astF.params.length; i++){
        r.push(astF.params[i].name);
      }
      return r;
    },
    findAllVariablesDeclaredOnPage(){
      var r = [];
      let ast = a.parse(this.code);
      for(let i=0; i<ast.body.length; i++){
        let el = ast.body[i];
        if(el.type==='VariableDeclaration'){
          console.log('declaration');
          var dec = this.getVariblesFromDeclaration(el.declarations);
          this.appendOnArray(r, dec);
        }
      }
      return r;
    },
    getVariblesFromDeclaration(declaration){
      var r = [];
      for(let i=0; i<declaration.length; i++)
        {
          console.log('variable');
          r.push(declaration[i].id.name);
        }
      return r;
    },
    appendOnArray(array, secound){
      for(let i=0; i<secound; i++)
        array.push(secound[i]);
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
.calculation{
  border-left: 3px solid blue;
  position: absolute;
  height: 400px;
  width: 900px;
  right: 10px;
  top: 100px;
}
</style>
