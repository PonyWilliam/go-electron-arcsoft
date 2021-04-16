<template>
    <div class="container">
      <img id="logo" src="http:///arcsoft.dadiqq.cn/electron/logo.png" alt="">
      <img src="http:///arcsoft.dadiqq.cn/electron/arcsoft.png" alt="" id="arcsoft">
        <div class="people" v-if="people">
          <div class="background"></div>
          <img class="image" :src="`http://arcsoft.dadiqq.cn/face/${info.ID}.png`" />
          <div class="name">{{this.info.Name}}</div>
          <div class="info">
              <div class="info-title">员 工 信 息</div>
              <div class="item"><span>id: </span>{{this.info.ID}}</div>
              <div class="item"><span>工号: </span>{{this.info.Nums}}</div>
              <div class="item"><span>性别: </span>{{this.info.Sex}}</div>
              <div class="item"><span>电话: </span>{{this.info.Telephone}}</div>
              <div class="item"><span>描述: </span>{{this.info.Description}}</div>
          </div>
          <h1 class="item-title">所 借 物 品</h1>
          <div class="products">
              <div class="product" v-for="item in pinfo" :key="item.id">
                  <p class="p-item">名称:{{item.product_name}}</p>
                  <p class="p-item">ID:{{item.id}}</p>
                  <p class="p-item">是否重要:{{item.is_important==1?'是':'否'}}</p>
              </div>
          </div>
          <div class="submit">
            <button @click="submit">确认</button>
            <button @click="cancel">取消</button>
          </div>
        </div>
        <div class="nopeople" v-else>
            <div class="wait">
                等 待 借 出
            </div>
        </div>
    </div>
</template>
<script>
let url = "http://192.168.130.209:8080/"//单界面不再单独写配置文件了
let ws = new WebSocket("ws://localhost:7777/ws");
export default{
  data(){
    return{
        people:false,//判断是否有人来了
        imgsrc:'https://hbimg.huabanimg.com/97ce6a7f948b4959d3222736b7a49f3559c13e792ae47-GHwI0F_fw658/format/webp',
        info:{
            // ID:2,
            // Description:'',
            // Name:'',
            // Sex:'',
            // Telephone:'',
            // Nums:'',
        },
        pinfo:[],
        token:'',
        time:'',
        rfids:[],
        socket:false,
        full:false,
    }
  },watch:{
    rfids(val,oldval){
        if(val == [] || val == undefined){
            this.people = false
            return
        }else{
            this.people = true
        }
        for (let x of val){
             //请求地址
            this.getProductInfo(x)
            console.log(x)
        }
    }
  },
  mounted(){
      //初始化

    this.getToken()//获取token
    let that = this
    let temparr = []
    ws.onopen = function(evt) { 
        console.log("Connection open ...");
    };
    ws.onmessage = function(evt) {
        if(evt.data.substr(0,1) == "w"){
            //之后传入wid
            let wid = evt.data.substr(1,evt.data.length - 1)
            console.log(wid)
            fetch(`${url}work/worker/${wid}`,{
                "method":"GET"
            }).then(data=>data.json()).catch(err=>console.log(err)).then(res=>{
                that.info = res.data.worker
            })

        }
        else if(evt.data == "start"){
            that.socket = true
        }else if(evt.data == "end"){
            that.socket = false
            //处理完成，回复ok测试
            that.full = true
            that.rfids = temparr//避免重复刷新
            //返回信息测试
        }else{
            if(that.socket){
                let temp = true;
                for(let x of that.rfids){
                    if (x == evt.data){
                        temp = false
                        break
                    }
                }
                if (temp){
                    console.log(evt.data)
                    temparr.push(evt.data)
                }
            }
        }
    };

    ws.onclose = function(evt) {
        console.log("Connection closed.");
    };
  },
  methods: {
        getToken(){
        let formData = new FormData()
        formData.append("username","admin")
        formData.append("password","admin")
        fetch(`${url}work/login`,{
                  "method":"POST",
                  "body":formData,
              }).then(res=>res.json()).catch(err=>console.log(err)).then(data=>{
                  //获取到token
                  if(data.token){
                      console.log(data)
                      this.token = data.token
                      this.time = Date.parse(new Date()) / 1000
                  }
              })
      },
      getProductInfo(rfid){
          fetch(url + `product/rfid/${rfid}`,{}).then(data=>data.json()).catch(err=>console.log(err)).then(data=>{
                  //把数据放到对应位置
                  console.log(data)
                  this.pinfo.push(data)//丢进数组
          })
      },
      submit(){
          //确认提交 s
          //1. 立马切换到无人状态
        let wid = this.info.ID
        let form = new FormData()
        form.append("wid",wid)
        for(let x of this.pinfo){
            fetch(`${url}work/borrow/${x.id}`,{
                method:'POST',
                body:form,
                headers:new Headers({
                    Authorization:this.token
                })
            }).then(res=>res.json()).catch(err=>console.log(err)).then(res=>{
                    console.log(res)
            })
        }
        this.people = false
        //提交到服务器
        ws.send('ok')
        this.people = false
        this.rfids.length = 0
        this.info.length = 0
        this.pinfo.length = 0
        this.info = {}
      },
      cancel(){
          //取消提交，返回到程序
        ws.send('cancel')
        this.people = false
        this.rfids.length = 0
        this.info.length = 0
        this.info = {}
        this.pinfo.length = 0
      }
  }
}
</script>
<style lang="scss">
    .nopeople{
        width:100%;
        height:100%;
        position:fixed;
        top:50%;
        left:0;
    }
    .wait{
        font-size:80rpx;
        color:white;
        line-height:120rpx;
    }
    #logo{
        position: absolute;
        width:100px;
        height:60px;
        top:20px;
        right:20px;
        opacity: 0.75;
        z-index:3;
    }
    #arcsoft{
      position: absolute;
        width:120px;
        height:60px;
        top:20px;
        left:20px;
        opacity: 0.95;
        z-index:3;
    }
    .container{
        width: 100%;
        padding:0!important;
        text-align: center;
        background:url(http:///arcsoft.dadiqq.cn/electron/1.png) no-repeat fixed;
        background-size: auto;
        min-height:1080px;
        .background{
          width:100%;
          height:180px;
          background:url('https://ai.arcsoft.com.cn/resource/images/technology/banner/faceContrast.jpg') no-repeat fixed;
          background-size: 100% 180px;
          opacity: 1;
          z-index:2;
        }
        .name{
            margin:5px 0 20px 0;
            color:#eee;
            font-size:26px;
            line-height: 40px;
        }
        .info{
            width:95%;
            display: flex;
            justify-content: left;
            margin-left:2.5%;
            box-sizing: border-box;
            flex-direction: column;
            text-align: left;
            background:rgba(255,255,255,.75);
            backdrop-filter: blur(5px);
            padding:15px 20px;
            border-radius: 10px;
        }
        .info .info-title{
            font-size:22px;
            color:#111;
            font-weight: 200;
        }
        .info .item{
            font-size: 16px;
            line-height: 32px;
        }
        .title{
          font-size:28px;
          font-weight: 500;
          line-height: 45px;
        }
        .image{
          margin-top:-50px;
          width:100px;
          height:100px;
          border-radius: 50px;
          z-index:3;
          box-shadow:6px 4px 3px rgba(112,112,112,.35),
          6px -4px 3px rgba(112,112,112,.35);
        }
        .item-title{
          font-size:24px;
          font-weight: 300;
          line-height: 40px;
          color:orange;
          text-align: center;
          display: block;
        }
        .products{
            width:95%;
            display: flex;
            box-sizing: border-box;
            margin-left:2.5%;
            flex-direction: row;
            flex-wrap: wrap;
            justify-content: space-between;
        }
        .product{
            box-sizing:border-box;
            text-align: left;
            width: 47%;
            padding:20px 10px;
            background:rgba(255,255,255,.65);

            margin:10px 0;
            border-radius: 15px;
        }
        .submit{
            display: block;
            margin-top:30px;
        }
        .submit button{
            display: inline-block;
            padding:15px 40px;
            margin:0 15px;
            border-radius: 20px;
            background:linear-gradient(to right,deepskyblue,blue);
            opacity: 0.85;
            color:white;
            font-size:30px;
            line-height:50px;
        }                           
    }
</style>