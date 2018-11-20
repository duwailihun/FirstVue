<template>
  <section>
    <x-header :left-options="{showBack:false}">中再百万医疗核保反射问卷</x-header>
    <card
      v-for="(question,index) in questions"
      v-show="question.show"
      :key="index">
      <div
        slot="header"
        class="title">{{ question.title }}</div>
       <!-- class="title">{{ index+1 }}. {{ question.title }}</div>-->


      <!-- 多选题 --> 
      <div
        slot="content"
        v-if="question.type=='checklist'">
        <div
          class="weui-cells__title"
          style="display: none;"/>
        <div class="weui-cells weui-cells_checkbox">
          <label
            v-for="(option,index) in question.options "
            :key="index"
            class="weui-cell weui-check_label vux-checklist-label-left">
            <div class="weui-cell__hd">
              <input
                type="checkbox"
                class="weui-check"
                @change="getNextQue(question,option,$event)"
                :value="option.key" :checked='option.selected'>
              <i class="weui-icon-checked vux-checklist-icon-checked"/>
            </div>
            <div class="weui-cell__bd">
              <p/> <span class="vux-label-desc">{{ option.inlineDesc }}</span>
            </div>
          </label>
        </div>
      </div>
      <!-- 单选题 -->
      <div
        slot="content"
        v-else-if="question.type=='radio'">
        <div
          class="weui-cells__title"
          style="display: none;"/>
        <div class="weui-cells weui-cells_radio">
          <label
            v-for="(option,index) in question.options " :key="index"
            class="weui-cell weui-cell_radio weui-check__label">
            <div class="weui-cell__bd">
              <p>
                <img
                  class="vux-radio-icon"
                  style="display: none;">
                <span class="vux-radio-label">{{ option.inlineDesc }}</span>
              </p>
            </div>
            <div class="weui-cell__ft">
              <input 
                type="radio"
                :name="question.id"
                class="weui-check"                
                @change="getNextQue(question,option,$event)"
                :value="option.key" :checked='option.selected'>
              <span class="weui-icon-checked"/>
            </div>
          </label>
        </div>
      </div>
      <!-- 填空题 -->
      <group
        slot="content"
        v-else>
        <x-input
          placeholder="请填写内容"
          class="weui-vcode">
          <x-button
            slot="right"
            @click.native="getNextQue(question,null,null)"
            type="primary"
            mini>提交</x-button>
        </x-input>
      </group>
    </card>
    <div> <loading v-model="showLoading" :text="loadText"></loading></div>
    <p></p>
    <x-button text="提交核保"  type="primary"  @click.native="submitQue()"></x-button>
    
    <checklist
      v-show="false"
      slot="content"
      :max="1 "
      required
      :options="[]" />
  </section>
</template>

<script>
import {
  Group,
  Cell,
  XHeader,
  Card,
  Checklist,
  Radio,
  XInput,
  XButton,
  Loading,
  TransferDomDirective as TransferDom 
} from "vux";
import Vue from 'vue';

export default {
  name: "Questionnaire",
  components: {
    Group,
    Cell,
    XHeader,
    Card,
    Checklist,
    Radio,
    XInput,
    XButton,
    Loading
  }, directives: {
    TransferDom
  },
  data() {
    return {
      questions: [
      ],
      showLoading: false,
      loadText: 'Loading'
    };
  },
  methods: {
    getNextQue(question, option,$event) {
    //自定义终止异常 foreach 跳出循环
    var  BreakException= {};     
    //如果类型为radio
    if(question.type==='radio'){
        //查看前面的题是否回答完整
        if(this.isHaveNoAnswerQue(question)==true){
          this.alertError("请顺序答题！");
          if($event!=null){
            $event.target.checked=false;
          }
          return false;
        }
      
        var selectedKey = option.key;//选择的option的值
        //后面触发的子问题标志
        var par_refOption = question.refOption+'##'+question.id+'@@'+selectedKey;
        var isRequest = false; //是否请求过。
        //1 更改选中状态
        question.options.forEach((item,i) => {  
          if(item.key==selectedKey){
            item.selected=true;
          }else{
            item.selected=false;
            //这个option触发的问题全隐藏
            var hide_refOption = question.refOption+'##'+question.id+'@@'+item.key;
            this.questions.forEach((que,i) => {
            var refOption = que.refOption+''; 
              if(refOption.indexOf(hide_refOption) == 0){
                   que.show=false;
              }
            });
          }
        });         
        //选中的option触发的问题显隐控制
         this.questions.forEach((que,i) => {
           var refOption = que.refOption+''; 
           if(refOption.indexOf(par_refOption) == 0){
                //1.设置成显示
                que.show=true;
               //如果有子级问题，则不用调用接口
                isRequest = true;
              //2.直属父级是否选中触发此问题的option
              this.questions.forEach((ques,m)=>{
                    var str = que.refOption+'';
                    var str_arry  = str.substr(str.lastIndexOf("##")+2,str.length).split('@@');
                    var p_queID = str_arry[0];
                    var p_optvalue = str_arry[1];
                    if(ques.id==p_queID){
                       //查看option是否选中
                       var option_arry = ques.options.filter(item=>{
                              if(item.key== p_optvalue && item.selected==true){
                                return true;
                              }
                            });
                      //如果没选中，则隐藏                 
                      if(option_arry.length==0){
                          que.show=false;
                      }                        
                    }
              });

              if(que.show==true){
               var par_arry = refOption.split('##');//父问题数组
               par_arry.forEach((item,k)=>{
                     if(k>0){
                       var queID = item.split('@@')[0]; 
                       //通过问题ID找到问题
                       var flag = false;
                       this.questions.filter((ques,ind)=>{
                        if(ind>0){
                          if(ques.id===queID){
                                var str = ques.refOption+'';
                                var str_arry  = str.substr(str.lastIndexOf("##")+2,str.length).split('@@');
                                var p_queID = str_arry[0]; 
                                var p_optvalue = str_arry[1]; 
                                //找到父问题的父问题，查看option是否选中 
                                this.questions.filter(item=>{
                                  if(item.id==p_queID){
                                   var opt_arry =  item.options.filter(opt=>{
                                          if(opt.key== p_optvalue && opt.selected==true){
                                            return opt;
                                          }
                                        },p_optvalue);
                                    if(opt_arry.length==0){
                                       que.show=false;
                                       flag = true;
                                    } 
                                     return item;
                                  }
                                });
                          }
                        }
                      });
                      if(flag==true){
                        //  break;
                      }
                     }
               });
              }
           }
        });
        //去请求下一题服务器
        if(isRequest==false){
             this.requestServer(question,option);
        }
      //文本框
      }else if(question.type==='textarea'){
           //查看前面的题是否回答完整
            if(this.isHaveNoAnswerQue(question)==true){
              this.alertError("请顺序答题！");
              return false;
            }
             //选中
            question.options[0].selected=true;
            //判断下一题是否存在
           var isRequest = false;
           var par_refOption = question.refOption+'##'+question.id+'@@'+question.options[0].key;
           this.questions.forEach((que,i) => {
           var refOption = que.refOption+''; 
           if(refOption.indexOf(par_refOption) === 0){
                isRequest = true;
                //break;
           }});
           if(isRequest==false){
             this.requestServer(question,option);  
           }
      //checklist
      }else if(question.type==='checklist'){
         //查看前面的题是否回答完整
            if(this.isHaveNoAnswerQue(question)==true){
              this.alertError("请顺序答题！");
              return false;
            }
         //option是选中还是不选中
         var flag;
        try {
          question.options.forEach((opt,i)=>{
             if(opt.key===option.key){
                  if(opt.selected==true){
                      opt.selected=false;
                      flag = false;
                  }else{
                      opt.selected=true;
                      flag = true;
                  }
                  throw BreakException;
             }
         });
        } catch (error) {
          if (error !== BreakException) throw error;
        }    
         //选中，触发下一个问题
         if(flag==true){
           var isRequest = false;
           var par_refOption = question.refOption+'##'+question.id+'@@'+option.key;
          
           //选中的option触发的问题显隐控制
           this.questions.forEach((que,i) => {
           var refOption = que.refOption+''; 
           if(refOption.indexOf(par_refOption) === 0){
                //1.设置成显示
                que.show=true;
               //如果有子级问题，则不用调用接口
                isRequest = true;
              //2.直属父级是否选中触发此问题的option
              this.questions.forEach((ques,m)=>{
                    var str = que.refOption+'';
                    var str_arry  = str.substr(str.lastIndexOf("##")+2,str.length).split('@@');
                    var p_queID = str_arry[0];
                    var p_optvalue = str_arry[1];
                    if(ques.id===p_queID){
                           ques.options.forEach((opt,j)=>{
                             if(opt.key===p_optvalue){
                               if(opt.selected==false){
                                  que.show=false;
                                 //break;
                               }
                             }
                           });   
                    }
              });
              if(que.show==true){
               var par_arry = refOption.split('##');//父问题数组
               par_arry.forEach((item,k)=>{
                     if(k>0){
                       var queID = item.split('@@')[0];  
                       //通过问题ID找到问题
                       var flag = false;
                       this.questions.forEach((ques,m)=>{
                         if(m>0){
                            if(ques.id===queID){
                               
                                var str = ques.refOption+'';
                                var str_arry  = str.substr(str.lastIndexOf("##")+2,str.length).split('@@');
                                var p_queID = str_arry[0];  
                                var p_optvalue = str_arry[1];
                                this.questions.forEach((questi,n)=>{
                                  if(questi.id===p_queID){
                                     questi.options.forEach((opt,j)=>{
                                     if(opt.key===p_optvalue){
                                      if(opt.selected==false){//如果没选中
                                        que.show=false;
                                        flag = true;
                                        //  break;
                                      }
                                      }
                                  });
                                  }
                                });    
                            }
                         }
                       });
                       if(flag==true){
                        //  break;
                       }
                     }
               });
              }
           }
          });
           if(isRequest==false){
              this.requestServer(question,option);           
           }

         //勾掉，隐藏其触发的问题
         }else{
             //这个option触发的问题全隐藏
            var hide_refOption = question.refOption+'##'+question.id+'@@'+option.key;
            this.questions.forEach((que,i) => {
            var refOption = que.refOption+''; 
              if(refOption.indexOf(hide_refOption) === 0){
                   que.show=false;
              }
            });
         }
      }  
     },getCommonParam(){
        let sex = this.$route.query.sex; //接受url地址中的被保险人性别
        let caseNo = this.$route.query.caseNo; //接受url地址中的投保单号
        let age = this.$route.query.age; //接受url地址中的被保险人年龄
        let riskType = this.$route.query.riskType; //接受url地址中的投保险类
        let insuCom = this.$route.query.insuCom; //接受url地址中的公司编码
        let token = this.$route.query.token; //接受url地址中的token
        // let token ='be662c61f7a14f751a5991e1647ea1eb';
        return {sex:sex,caseNo:caseNo,age:age,riskType:riskType,insuCom:insuCom,token:token};
     },isHaveNoAnswerQue(question){
      var index = this.getIndex(question);
      var beforQuesions = this.questions.filter((que,i)=>{
          if(que.show==true && i<index ){
            return true;
          }
        });
        var ques =  beforQuesions.filter((que,i)=>{
            var flag =false;
            var beforopts =  que.options.filter(opt=>{
                if(opt.selected==true){
                  flag=true; 
                  return true;
                }
             });
             if(flag==false){
               return true;
             }
        });        
        if(ques.length>0){
           return true;
        }else{
          return false;
        }

     },requestServer(question,option){
             this.showLoading=true;
             var optvalue ;
             if(option==null){
               optvalue = '01';
             }else{
               optvalue = option.key;
             }
             //设置成selected=true 保持一致
             var param = this.getCommonParam();
             param['questionCode']=question.id;
             param['questionAnswerValue']=optvalue;
             param['questionnaireId']=question.questionnaireId;
           //2 请求服务追加下一问题
            this.$http.get('/api/interfaceAdapter/getNextQuestion',{params:param}).then(result=>{
              this.showLoading=false;
              var result=result.data;
                result.refOption=question.refOption+'##'+question.id+'@@'+optvalue; 
                result.show=true;
                if(result.type==='textarea'){
                  result.options.push({key:'01',inlineDesc: 'defaultName'});
                }  
                //1.参数非法
                if(result.errMessage!=' '){
                  this.alertError(result.errMessage);
                  return ;
                }
                //2.不是最后一题
                if(result.questionnaireId!=' '){
                  //如果是第一题则追加到最后面
                 if(question.refOption==''){
                   this.questions.push(result);
                 }else{
                   //追加到触发option的后面
                   var index = this.getIndex(question);
                   if(index==-1){
                     this.alertError('追加问题时，数组中未找到父问题index');
                   }else{
                       this.questions.splice(index+1,0,result);  
                   }
                 }
                }
              }).catch(error=> {
                  console.log('发生错误：'+error);
                  this.showLoading=false;   
                  this.alertError();    
              });
     //提交表单
     },getIndex(question){
      var index = -1;
      this.questions.forEach((item,i)=>{
        if(item.id === question.id){
          index = i;
        }
      });
      return index ;
    }
    ,submitQue(){
       this.showLoading=true;        
       //没有回答完全
       var reqJson=[];
      var show_que_arry =  this.questions.filter(question=>{
          if(question.show==true){
            return question;
          }
       });
         var flag =false;
         show_que_arry.forEach((que,i)=>{
          var opt_arry = que.options.filter(opt=>{
             if(opt.selected==true){
               return true;
             }
          });
          if(opt_arry.length==0){
            flag=true;
            //break;
          }
         });
         if(flag==true){
           this.showLoading=false;        
           this.alertError('您还有题目未回答！');
           return;
         }
     
        //第一题有几个选项选中
        // var firstQue = show_que_arry[0];
        // var firstSelectedOpt = firstQue.options.filter(opt=>{
        //    if(opt.selected==true){
        //      return true;
        //    }
        // });
        //  firstSelectedOpt.forEach((opt,i)=>{
        //    reqJson.push({
        //         QuestionnaireId:firstQue.questionnaireId
        //        ,QuestionCode:firstQue.id
        //        ,QuestionAnswerCode:firstQue.id
        //        ,QuestionAnswerValue:opt.key
        //    });
        //    //遍历显示的question数组，拼接问卷ID和opt.key相等的问题 
        //       var val = opt.key;
        //       var arry =  show_que_arry.filter(que=>{
        //           if(que.questionnaireId==val){
        //             return true;
        //           }
        //       },val);
        //       arry.forEach(que => {
        //         var opt_arry = que.options.filter(opt=>{
        //                 if(opt.selected==true){
        //                   return true;
        //                 }
        //         });
        //         //多个选项用逗号分隔
        //         var str ='';
        //         if(opt_arry.length>1){
        //             opt_arry.forEach(ele => {
        //                  str = ele.key+'%26';
        //             });
        //             str =str.substring(0,str.length-3);
        //         }else{
        //           str=opt_arry[0].key;
        //         }
        //         reqJson.push({
        //           QuestionnaireId:val 
        //           ,QuestionCode:que.id
        //           ,QuestionAnswerCode:que.id
        //           ,QuestionAnswerValue:str
        //         });

        //       });             
        //   });
          
          show_que_arry.forEach(que => {
                var opt_arry = que.options.filter(opt=>{
                        if(opt.selected==true){
                          return true;
                        }
                });
                //多个选项用逗号分隔
                var str ='';
                if(opt_arry.length>1){
                    opt_arry.forEach(ele => {
                         str = ele.key+'%26';
                    });
                    str =str.substring(0,str.length-3);
                }else{
                  str=opt_arry[0].key;
                }
                reqJson.push({
                  QuestionnaireId:val 
                  ,QuestionCode:que.id
                  ,QuestionAnswerCode:que.id
                  ,QuestionAnswerValue:str
                });
              });             
          var qs = require('qs');
           var  param = this.getCommonParam();
           param['questionAnswerValue']= JSON.stringify(reqJson);
         
          this.$http.post('/api/interfaceAdapter/submissionSelfCore',
          qs.stringify(param,{ indices: false })).then(result=>{
                 console.log(result.data);
                 this.showLoading=false;        
                 window.location.href=result.data;
          }).catch(error=> {
                  console.log('发生错误：'+error);
                  this.showLoading=false;     
                  this.alertError();  
          });
         
     },
     alertError(info){
        if(info==null){
          info='服务异常，请稍后重试!';
        }
        this.$vux.alert.show({
          title: info,
          content: ''
        });
     }
    },
  mounted() {   
    var  param = this.getCommonParam();
     //显示loading       
    this.showLoading=true;         
    //进入页面后，第一次请求
    this.$http.get('/api/interfaceAdapter/Adapter', {params: param
        }).then(result=>{
      //隐藏loading
      this.showLoading=false;
      var result =result.data;
      //添加父节点标识
      result.refOption=''; 
      result.show=true;
      if(result.type==='textarea'){
        result.options.push({key:'01',inlineDesc: 'defaultName'});
      }
      //添加到questions
      this.questions.push(result);
    }).catch(error=> {
      console.log('发生错误：'+error);
      this.showLoading=false;   
      this.alertError();  
    });
 
  }
};
</script>
<style lang="less">
@import "~vux/src/styles/1px.less";
.title {
  //font-size: 0.9375rem;
  font-size: 0.8125rem;
  color: #333333;
  padding-left: 1.25rem;
  line-height: 2.1875rem;
}
.vux-header .vux-header-title{
  font-size: 1rem !important;
}
.vux-header{
  background-color:#4286f5 !important;
}
.weui-cells {
  margin-top: 0 !important;
  font-size: 0.8125rem !important;
  color: #333333;
} 
.card-padding {
  padding: 0.9375rem;
}
.weui-panel:before {
  border-top: none !important;
}
.weui-btn_primary{
  background-color:#4286f5 !important;
}
.weui-btn{
  font-size:0.8125rem !important;
}
.weui-cells_checkbox .weui-check:checked+.weui-icon-checked:before {
    content: "\EA06";
    color: #4286f5 !important; 
}
</style>