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
      <b-modal ref="myModalRef" id="myModal" @ok='generateProgram'>
        <ul>
          <li>Arguments</li>
          <li v-for="(v, index) in functionData.arguments" :key="v.name">
            {{ v.name }} <input type='text' v-model='v.value' />
          </li>
          <li>Outscoper</li>
          <li v-for="(v, index) in functionData.outscoper" :key="v.name">
            {{ v.name }} <input type='text' v-model='v.value' />
          </li>
        </ul>
      </b-modal>
      <b-modal ref="myModalRef2" id="myModal2" >
        <h2>Test function</h2>
        <pre class='function-content'>
          {{testFunction}}
        </pre>
      </b-modal>
  </div>
</template>

<script>
import VueObjectView from 'vue-object-view';
import acorn from "acorn";
let a = require('acorn');
import estraverse from "estraverse";
import escodegen from "escodegen";
import esprima from "esprima";
import BootstrapVue from 'bootstrap-vue';
import 'bootstrap/dist/css/bootstrap.css';
import 'bootstrap-vue/dist/bootstrap-vue.css';

let escope = require('escope');
import Vue from 'vue'
Vue.use(BootstrapVue);
export default {
  components:{
    "VueObjectView": VueObjectView,
  },
  data() {
    return {
      code: 'let a = 5;',
      object: {},
      see: true,
      see2: true,
      calculation: {},
      functionData: {},
      functionAst: null,
      functionName: 'ispravna',
      testFunction: '',
    }
  },
  methods: {
    showModal () {
      this.$refs.myModalRef.show()
    },
    hideModal () {
      this.$refs.myModalRef.hide()
    },
    showModal2 () {
      this.$refs.myModalRef2.show()
    },
    hideModal2 () {
      this.$refs.myModalRef2.hide()
    },
    setArrayToFunctionObject(array){
      let r = [];
      for(let i=0; i<array.length; i++){
        r.push({name: array[i], value: ''});
      }
      return r;
    },
    consolelog(){
      let node = a.parse(this.code);
      console.log(node);
    },
    run(){
      this.object = esprima.parse(this.code);
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
      let ast = esprima.parse(this.code);
      let myFunction = this.findFunction(ast, this.functionName);
      let usedVariables = this.findAllVariables(myFunction);
      this.functionAst = myFunction;

      let scopeManager = escope.analyze(ast);
      let funScope = this.getFunctionScope(scopeManager, "ispravna");
      // console.log(funScope);
      let funScopeVariables = this.getDeclardVariablesFromScope(funScope);
      let funArguments = this.getFunArguments(funScope);

      let funUsdVarOutScope = this.setsDifference(usedVariables, funScopeVariables); // Variables that function used from outside scoper

      // console.log(usedVariables);
      // console.log(funScopeVariables);
      // console.log(funUsdVarOutScope);
      // console.log(funArguments);

      this.functionData = {
        arguments: this.setArrayToFunctionObject(funArguments),
        outscoper: this.setArrayToFunctionObject(funUsdVarOutScope)
      };
      this.showModal();

      // this.generateProgram();
    },
    generateProgram(){
      let startNode = {
        type: "Program",
        body: [],
      };

      let dec = {
        type: "VariableDeclaration",
        declarations: [],
        kind: "var",
      };
      // Adding outscope variables
      for(let i=0; i<this.functionData.outscoper.length; i++){
        let var1 = {
          type: "VariableDeclarator",
          id: {
            type: "Identifier",
            name: this.functionData.outscoper[i].name
          },
          init: {
            type: "Literal",
            value: this.functionData.outscoper[i].value
          }
        }
        dec.declarations.push(var1);
      }
      if(this.functionData.outscoper.length>0)
        startNode.body.push(dec);
      startNode.body.push(this.functionAst);
      // caling function
      let fcal = {
        type: "ExpressionStatement",
        expression: {
          type: "CallExpression",
          callee: {
            type: "Identifier",
            name: this.functionName
          },
          arguments: []
        }
      }
      let a = fcal.expression.arguments;
      for(let i=0; i<this.functionData.arguments.length; i++){
        let ar = {
          type: "Literal",
          value: this.functionData.arguments[i].value
        }
        a.push(ar);
      }
      startNode.body.push(fcal);
      this.testFunction = escodegen.generate(startNode);
      this.showModal2();
    },
    // ESPRIMA CODE
    findAllVariables(ast){
      let result = [];
      let vue = this;
      estraverse.traverse(ast, {
        enter: function (node, parent) {
          if(node.type==='CallExpression'){
            for(let i=0; i<node.arguments.length; i++)
              vue.addToSet(result, node.arguments[i].name);
            return estraverse.VisitorOption.Skip;
          }
          if(node.type==='Property'){
            if(node.value.type ==='Identifier')
              vue.addToSet(result, node.value.name);
            return estraverse.VisitorOption.Skip;
          }
          if(node.type==='MemberExpression'){
            vue.addToSet(result, node.object.name);
            return estraverse.VisitorOption.Skip;
          }
        },
        leave: function (node, parent) {
          if (  
            node.type == 'Identifier' && 
            (parent===null || parent.type !== 'VariableDeclarator') &&  // Not decalared variable
            (parent.type !== 'FunctionDeclaration') // Not function declaration
          )
          vue.addToSet(result, node.name);
        }
      });
      return result;
    },
    findFunction(ast, name){
      var r = null;
      estraverse.traverse(ast, {
        enter(node, parent) {
          if(node.type==='FunctionDeclaration' && node.id.name===name){
            r = node;
            return estraverse.VisitorOption.Break;
          }
        }
      });
      return r;
    },
    getFunctionScope(scopeManager, funName){
      let s = scopeManager.scopes;
      for(let i=0; i<s.length; i++){
        let scope = s[i];
        if(scope.type!=='function')
          continue;
        if(scope.block.id.name === funName)
          return scope;
      }
      return null;
    },
    getDeclardVariablesFromScope(scope){
      let result = [];
      for(let i=1; i<scope.variables.length; i++)
        result.push(scope.variables[i].name);
      return result;
    },
    getFunArguments(scope){
      var r = [];
      for(let i=0; i<scope.block.params.length; i++)
        r.push(scope.block.params[i].name);
      return r;
    },
    // ACORNE CODE
    findAllFunctions(ast){
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
    findAllVariablesDeclaredOnPage(ast){
      var r = [];
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
      for(let i=0; i<declaration.length; i++){
        r.push(declaration[i].id.name);
      }
      return r;
    },
    getUsedVariablesInFunction(funAst){
      let body = funAst.body.body;
      for(let i=0; i<body.length; i++){

      }
    },
    getUsedVariableFromNode(nodeAst){
      
    },
    // UTILITY FUNCTIONS
    appendOnArray(array, secound){
      for(let i=0; i<secound.length; i++)
        array.push(secound[i]);
    },
    addToSet(array, value){
      for(let i=0; i<array.length; i++){
        if(array[i]==value){
          return false;
        }
      }
      array.push(value);
      return true;
    },
    setsDifference(array1, array2){ // All elements from array1, that are not present in array2
      let r = [];
      for(let i=0; i<array1.length; i++){
        if( !this.contanis(array2, array1[i]) )
          r.push(array1[i]);
      }
      return r;
    },
    contanis(array, value){
      for(let i=0; i<array.length; i++)
        if(array[i]===value)
          return true;
      return false;
    }
  },
  created(){
    this.object = a.parse(this.code);
    console.log(a);
    console.log(escope);
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
  max-height: 600px;
  width: 900px;
  right: 10px;
  top: 100px;
  overflow: scroll;
}
.function-content{
  height: 400px;
  width: 600px;
}
</style>
