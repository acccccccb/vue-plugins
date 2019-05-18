<template>
  <div>
    <el-button type="primary" @click="visible=true">选择裁剪</el-button>
    <el-dialog
      title="裁剪图片"
      :visible.sync="visible"
      width="943px"
      :before-close="handleClose">
      <div class="toolMain">
        <div>offsetX:{{mouseMove.x}}  offsetY:{{mouseMove.y}}</div>
        <div>offsetX:{{mouseDown.x}}  offsetY:{{mouseDown.y}}</div>
        <div>offsetX:{{mouseUp.x}}  offsetY:{{mouseUp.y}}</div>
        <div ref="toolBox" class="toolBox" v-on:mousemove="handleMouseMove" v-on:mousedown="handleMouseDown" v-on:mouseup="handleMouseUp">
          <div
            v-on:mousedown="controlBoxMouseDown"
            v-on:mouseup="controlBoxMouseUp"
            v-on:mousemove="controlBoxMouseMoveEvent"
            ref="toolBoxControl"
            class="toolBoxControl">
            <div class="controlBox">
              <div class="leftUp controleBtn"></div>
              <div class="leftDown controleBtn"></div>
              <div class="rightUp controleBtn"></div>
              <div class="rightDown controleBtn"></div>
            </div>
          </div>
          <canvas class="canvasSelectBox" ref="canvasSelectBox" :width="boxWidth" :height="boxHeight"></canvas>
          <canvas class="canvas" ref="canvas" :width="boxWidth" :height="boxHeight"></canvas>
        </div>
        <div class="toolBar">
          <input v-show="false" @change="putImgToCanv" ref="inputFile" type="file" accept="image/*">
          <el-button size="mini" @click="chooseImg" type="primary" plain>选择图片</el-button>
        </div>
      </div>
      <span slot="footer" class="dialog-footer">
    <el-button @click="handleClose">取消</el-button>
    <el-button type="primary" @click="handleClose">确定</el-button>
  </span>
    </el-dialog>
  </div>
</template>
<script>
  export default {
    name:'CropTool',
    data(){
      return {
        visible:true,
        boxWidth:900,
        boxHeight:450,
        offset:0,
        mouseMove:{
          x:0,
          y:0,
        },
        mouseDown:{
          x:0,
          y:0
        },
        mouseUp:{
          x:0,
          y:0
        },
        rate:null,
        drawImg:{
          img:null,//规定要使用的图像、画布或视频
          sx:0,//开始剪切的 x 坐标位置
          sy:0,//开始剪切的 y 坐标位置
          swidth:0,//被剪切图像的宽度
          sheight:0,//被剪切图像的高度
          x:0,//在画布上放置图像的 x 坐标位置
          y:0,//在画布上放置图像的 y 坐标位置
          width:0,//要使用的图像的宽度
          height:0//要使用的图像的高度
        },
        selectBox:false,
        selectBoxColor:'rgba(0,0,0,0.4)',
        selectBoxOriginalTop:0,
        selectBoxOriginalLeft:0,

        controlBoxMouseMove:false,
        controlBoxStartX:0,
        controlBoxStartY:0,
        controlBoxOriginalTop:0,
        controlBoxOriginalLeft:0,
      }
    },
    methods:{
      handleClose:function(){
        this.visible = false;
      },
      handleMouseDown:function(e){
        this.mouseDown.x = e.offsetX;
        this.mouseDown.y = e.offsetY;
        this.selectBox = true;
        let $toolBoxControl = this.$refs['toolBoxControl'];
        $toolBoxControl.style.pointerEvents = 'none';
      },
      handleMouseUp:function(e) {
        this.mouseUp.x = e.offsetX;
        this.mouseUp.y = e.offsetY;
        this.selectBox = false;
        let $toolBoxControl = this.$refs['toolBoxControl'];
        $toolBoxControl.style.pointerEvents = 'auto';
      },
      handleMouseMove:function(e){
        this.mouseMove.x = e.offsetX;
        this.mouseMove.y = e.offsetY;
        if(this.selectBox && this.drawImg.img) {
          this.drawChooseArea();
          this.drawToolBox();
        }
      },
      controlBoxMouseDown:function(e){
        console.log('控制区按下');
        this.controlBoxMouseMove = true;
        this.controlBoxStartX = e.clientX;
        this.controlBoxStartY = e.clientY;
        let $toolBoxControl = this.$refs['toolBoxControl'];
        this.controlBoxOriginalTop = parseInt($toolBoxControl.style.top.split('px')[0]);
        this.controlBoxOriginalLeft = parseInt($toolBoxControl.style.left.split('px')[0]);
        e.stopPropagation();
      },
      controlBoxMouseUp:function(e){
        console.log('控制区抬起');
        this.controlBoxMouseMove = false;
        this.selectBoxOriginalTop = this.controlBoxOriginalTop;
        this.selectBoxOriginalLeft = this.controlBoxOriginalLeft;
        e.stopPropagation();
      },
      controlBoxMouseMoveEvent:function(e){
        if(this.controlBoxMouseMove) {
          console.log(e);
          let offsetX = e.clientX - this.controlBoxStartX;
          let offsetY = e.clientY - this.controlBoxStartY;
          let $toolBoxControl = this.$refs['toolBoxControl'];
          let top = this.controlBoxOriginalTop;
          let left = this.controlBoxOriginalLeft;
          console.log('offset',offsetX,offsetY);
          $toolBoxControl.style.top = (top+offsetY)+'px';
          $toolBoxControl.style.left = (left+offsetX)+'px';
          this.drawChooseArea(left+offsetX,top+offsetY);
        }
        e.stopPropagation();
      },
      // 选择图片
      chooseImg:function(){
        this.$refs['inputFile'].value = "";
        this.drawImg.img = null;
        this.$refs['inputFile'].click();
      },
      // 将选择的图片绘制到画布
      putImgToCanv:function(e){
        let _this = this;
        let file = e.target.files[0] || null;
        if(file) {
          let reader = new FileReader();
          let compressImg = new Image();
          reader.readAsDataURL(file);
          reader.onload = function(e) {
            // 图片base64化
            let newUrl = e.target.result;
            let img = document.createElement('img');
            img.src = newUrl;
            let timmer = setInterval(function(){
              if(reader.readyState == 2) {
                clearInterval(timmer);
                let imgHeight = img.height;
                let imgWidth = img.width;
                let boxWidth = _this.boxWidth;
                let boxHeight = _this.boxHeight;
                let c = _this.$refs['canvas'];
                let ctx = c.getContext("2d");
                let rate;
                let x=0,y=0;
                let drawImg = _this.drawImg;
                drawImg.img = img;
                if(imgHeight<boxHeight && imgWidth < boxWidth) {
                  rate = 1;
                  drawImg.x = (boxWidth - imgWidth)/2;
                  drawImg.y = (boxHeight - imgHeight)/2;
                } else {
                  if(imgWidth/imgHeight <= boxWidth/boxHeight) { // 计算宽高比
                    rate = boxHeight/imgHeight;
                    drawImg.x = (boxWidth - imgWidth*rate)/2;
                  } else {
                    rate = boxWidth/imgWidth;
                    drawImg.y = (boxHeight - imgHeight*rate)/2;
                  }
                }
                ctx.clearRect(0,0,c.width,c.height);
                drawImg.swidth = imgWidth;
                drawImg.sheight = imgHeight;
                drawImg.width = imgWidth*rate;
                drawImg.height = imgHeight*rate;
                ctx.drawImage(drawImg.img,drawImg.sx,drawImg.sy,drawImg.swidth,drawImg.sheight,drawImg.x,drawImg.y,drawImg.width,drawImg.height);
                (function(){
                  let c=_this.$refs['canvasSelectBox'];
                  let ctx=c.getContext("2d");
                  ctx.fillStyle = _this.selectBoxColor;
                  ctx.clearRect(0,0,c.width,c.height);
                  ctx.fillRect(0,0,c.width,c.height);
                })()
              }
            },200);
          };
        }
      },
      // 绘制选区
      drawChooseArea:function(originX,originY){
        let _this = this;
        let c=this.$refs['canvasSelectBox'];
        let ctx=c.getContext("2d");
        ctx.fillStyle = this.selectBoxColor;
        ctx.clearRect(0,0,c.width,c.height);
        ctx.fillRect(0,0,c.width,c.height);
        let clearWidth,clearHeight;
        if(!this.rate) { // 不限比例
          clearWidth = this.mouseMove.x-this.mouseDown.x;
          clearHeight = this.mouseMove.y-this.mouseDown.y;
        } else { // 等比例裁剪
          let rate = this.rate.split(':')[1] / this.rate.split(':')[0];
          clearWidth = this.mouseMove.x-this.mouseDown.x;
          clearHeight = clearWidth*rate;
        }
        _this.selectBoxOriginalLeft = this.mouseDown.x - this.offset||0;
        _this.selectBoxOriginalTop = this.mouseDown.y - this.offset||0;
        if(originX && originY) {
          console.warn(1);
          ctx.clearRect(originX,originY,clearWidth,clearHeight);
        } else {
          ctx.clearRect(_this.selectBoxOriginalLeft,_this.selectBoxOriginalTop,clearWidth,clearHeight);
        }
      },
      // 绘制选择框
      drawToolBox:function(){
        let clearWidth,clearHeight;
        if(!this.rate) { // 不限比例
          clearWidth = this.mouseMove.x-this.mouseDown.x;
          clearHeight = this.mouseMove.y-this.mouseDown.y;
        } else { // 等比例裁剪
          let rate = this.rate.split(':')[1] / this.rate.split(':')[0];
          clearWidth = this.mouseMove.x-this.mouseDown.x;
          clearHeight = clearWidth*rate;
        }
        // 绘制操作杆
        let $toolBoxControl = this.$refs['toolBoxControl'];
        $toolBoxControl.style.width = clearWidth+'px';
        $toolBoxControl.style.height = clearHeight+'px';
        $toolBoxControl.style.top = Math.abs(this.mouseDown.y - this.offset) + 'px';
        $toolBoxControl.style.left = Math.abs(this.mouseDown.x - this.offset) + 'px';
      },
    }
  }
</script>
<style scoped>
  .toolMain {
    height:500px;
    box-sizing: border-box;
  }
  .toolBox {
    width:900px;
    height:450px;
    border:1px solid #dedede;
    background:#dedede;
    position: relative;
  }
  .canvas {
    position: absolute;
    top:0;
    left:0;
    z-index:99;
  }
  .canvasSelectBox {
    position: absolute;
    top:0;
    left:0;
    z-index:100;
  }
  .toolBoxControl {
    background:transparent;
    position:absolute;
    z-index:101;
    box-sizing:border-box;
    border:1px dashed rgba(255,255,255,0.6);
  }
  .controlBox {
    width:100%;
    height:100%;
    position: absolute;
    cursor:move;
  }
  .controleBtn {
    width:10px;
    height:10px;
    box-sizing:border-box;
    border:1px solid #409EFF;
    background:#409EFF;
    position: absolute;
  }
  .leftUp {
    top:0;
    left:0;
    margin-left:-5px;
    margin-top:-5px;
    cursor:se-resize;
  }
  .leftDown {
    bottom:0;
    left:0;
    margin-left:-5px;
    margin-bottom:-5px;
    cursor:sw-resize;
  }
  .rightUp {
    top:0;
    right:0;
    margin-right:-5px;
    margin-top:-5px;
    cursor:sw-resize;
  }
  .rightDown {
    bottom:0;
    right:0;
    margin-right:-5px;
    margin-bottom:-5px;
    cursor:se-resize;
  }
  .toolBar {
    margin-top:20px;
  }
</style>
