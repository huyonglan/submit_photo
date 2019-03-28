<!-- 上传图片 -->
<template>
  <div class="submit-photo">
    <!-- 标题 -->
    <div class="top-title">
      <p>上传过程中离开，将终止上传进程(还能添加<span>{{allowNum}}</span>张图片)</p>
    </div>
    <!-- 照片 -->
    <div class="photo">
      <div class="photo-add" :class="{'hidden':allowNum==0}">
        <div class="add-img">+</div>
        <p>点击添加</p>
        <input v-if="!showTips" class="input-file" type="file" name="image" accept="image/*" ref="file"
               @change="handleUp"
               multiple="true"/>
      </div>

      <div v-for="(item,index) in images" class="photo-img">
        <img :src="item.url" @click="showBigImg(item.url)"/>
        <p>{{item.status===0?'上传中':showDate}}</p>
        <div class="img-del" @click="handlerDeletePhoto(index)">
          <svg aria-hidden="true">
            <use xlink:href="#icon-GroupCopy"></use>
          </svg>
        </div>
      </div>

    </div>


    <!--<p>-&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;&#45;</p>-->
    <!--<div class="photo">-->
    <!--<div v-for="(item) in anotherImg" class="photo-img">-->
    <!--<img :src="item"/>-->
    <!--</div>-->
    <!--</div>-->

    <!--按钮 -->
    <div class="btn">
      <button :class="{'disable':!isCanCommit}" @click="handlerSubmit">提交作业</button>
    </div>

    <canvas id="cvs"></canvas>

    <canvas id="wpcvs"></canvas>

    <!-- 预览图片 -->
    <wimg :show="isShowBigImg" :imgs="bigImages" :currentImg="currentImg" @close="handlerCloseBigImg"></wimg>

    <!-- 提示 -->
    <back-tip v-if="showTips" @cancel="handlerCancelTips" @save="handlerSaveTips"></back-tip>
  </div>
</template>

<script>
  import wimg from 'w-previewimg';

  import backTip from './components/back-tip';

  //接口
  import api from '../../api/api-view';

  export default {
    name: '',
    components: {
      wimg,
      backTip
    },
    data() {
      return {
        // TODO 最大图片
        imgMaxSize: 1024 * 1024 * 10,// 10M
        // 图片资源
        images: [],
        // 是否显示预览图片
        isShowBigImg: false,
        // 当前图片
        currentImg: '',
        // 允许上传的图片张数
        allowNum: 0,
        // 图片上传总张数
        allowTotal: 15,
        // 是否显示提示
        showTips: false,

        // 是否提交 防止重复提交
        isSubmit: false,
        // 返回次数
        backNum: 0,
        // 作业id
        eljobId: '',
        // 没图片时已经返回
        hadGone: false,
        // 测试图片
        //anotherImg:[]
      }
    },
    computed: {
      // 放大图片数组
      bigImages() {
        const images = this.images.map(res => {
          return res.url;
        });
        return images;
      },
      // 是否可以提交
      isCanCommit() {
        const list = this.images;
        if (list.length === 0) {
          return false;
        }

        for (let item of list) {
          if (!item.status) {
            return false;
          }
        }

        return true;
      },
      // 显示如期
      showDate() {
        const date = new Date();
        const month = date.getMonth() + 1;
        const day = date.getDate();

        return `${month}月${day}日`;

      }
    },
    methods: {
      // 上传图片
      handleUp() {
        const fileList = Object.values(this.$refs.file.files);
        // 超过应上传张数 自动截取
        if (fileList.length > this.allowNum) {
          fileList.splice(this.allowNum);
        }

        for (let val of fileList) {
          // 检查文件类型
          if (['jpeg', 'png', 'gif', 'jpg'].indexOf(val.type.split('/')[1]) < 0) {
            this.$toast("上传格式不符合要求");
            return;
          }

          // 检查文件大小
          if (val.size > this.imgMaxSize) {
            this.$toast("上传图片不能超过10MB");
            return;
          }
        }

        const that = this;

        // 循环取得图片的url
        fileList.forEach(async function (file, i) {
          const obj = {id: '', status: 0};
          obj.url = URL.createObjectURL(file);
          obj.name = file.name;
          that.images.push(obj);
          that.allowNum--;

          that.uploadPhoto(file, that.images.length - 1)
        });
      },

      // 发请求上传
      uploadPhoto(file, i) {
        const vm = this;
        const eljobId = vm.eljobId;
        const date = new Date();
        //console.log("===", this.images, i);
        this.images[i].date = date;

        let reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = function () {
          let img = new Image();
          img.src = this.result;

          img.onload = async () => {
            let Orientation = null;

            await EXIF.getData(img, function () {

              EXIF.getAllTags(this);

              Orientation = EXIF.getTag(this, 'Orientation');//这个Orientation 就是我们判断需不需要旋转的值了，有1、3、6、8

              console.log("---", Orientation)
            });
            // 压缩图片
            let pressedResult = vm.compress(img, file.type, Orientation, file.name);
            vm.images[i].url = pressedResult;
            img = null;
            axiosUpload(pressedResult);
          }
        };

        const axiosUpload = (result) => {
          const url = api.upload;
          // 发送消息
          this.$http.post(url, {
            fileName: file.name,
            eljobId,
            code: result.split(',')[1]
          }).then(res => {
            // 数组里去找对应date， 找到了 将返回的id插进去，并且更新status的值为1
            const index = this.images.findIndex(val => {
              return val.date == date;
            });
            // console.log("tempObj is", index);
            if (index === -1) {
              return;
            }
            // 找到之后 将后端返回的id插入对应的 数组元素上
            const tempObj = this.images[index];
            tempObj.id = res.data;
            tempObj.status = 1;
            this.$set(this.images, index, tempObj);
          });

        }

      },

      // 压缩图片
      compress(img, type, orientation, name) {
        let canvas = document.getElementById("cvs");
        let ctx = canvas.getContext('2d');

        let initSize = img.src.length;
        let width = img.width;
        let height = img.height;
        //如果图片大于四百万像素，计算压缩比并将大小压至400万以下
        let ratio;
        if ((ratio = width * height / 4000000) > 1) {
          ratio = Math.sqrt(ratio);
          width /= ratio;
          height /= ratio;
        } else {
          ratio = 1;
        }
        console.log("width, height, ratio", width, height, ratio);
        canvas.width = width;
        canvas.height = height;

        if (orientation && orientation != 1) {    //小米6竖着拍摄旋转问题
          console.log("amazing");
          switch (orientation) {
            case 6:     // 需要顺时针旋转90度
              this.rotateImg(img, 'left', canvas);
              break;
            case 8:     // 需要逆时针旋转90度
              this.rotateImg(this,'right',canvas);
              break;
            case 3:     // 需要180度旋转
              this.rotateImg(this,'right',canvas);//转两次
              this.rotateImg(this,'right',canvas);
              break;
          }
        } else {
          ctx.drawImage(img, 0, 0, width, height);
        }

        let ndata = '';
        if (initSize > 100 * 1024) {
          ndata = canvas.toDataURL(type, 0.1);
        } else {
          ndata = canvas.toDataURL(type, 0.8);
        }
        //进行最小压缩
        console.log('图片名称：'+name);
        console.log('压缩前：' + initSize);
        console.log('压缩后：' + ndata.length);
        console.log('压缩率：' + ~~(100 * (initSize - ndata.length) / initSize) + "%");
        canvas.width = canvas.height = 0;
        return ndata;
      },

      rotateImg (img, direction,canvas) {
        let min_step = 0;
        let max_step = 3;
        if (img == null)return;
        let height = img.height;
        let width = img.width;
        let step = 2;
        if (step == null) {
          step = min_step;
        }
        if (direction == 'right') {
          step++;
          step > max_step && (step = min_step);
        } else {
          step--;
          step < min_step && (step = max_step);
        }
        let degree = step * 90 * Math.PI / 180;
        let ctx = canvas.getContext('2d');
        switch (step) {
          case 0:
            canvas.width = width;
            canvas.height = height;
            ctx.drawImage(img, 0, 0);
            break;
          case 1:
            canvas.width = height;
            canvas.height = width;
            ctx.rotate(degree);
            ctx.drawImage(img, 0, -height);
            break;
          case 2:
            canvas.width = width;
            canvas.height = height;
            ctx.rotate(degree);
            ctx.drawImage(img, -width, -height);
            break;
          case 3:
            canvas.width = height;
            canvas.height = width;
            ctx.rotate(degree);
            ctx.drawImage(img, -width, 0);
            break;
        }
      },

      // 删除
      handlerDeletePhoto(index) {
        this.images.splice(index, 1);

        // 增加可上传张数
        this.allowNum = this.allowNum + 1;
      },

      // 显示预览图片
      showBigImg(val) {
        this.currentImg = val;
        this.isShowBigImg = true;
      },

      // 关闭预览图片
      handlerCloseBigImg() {
        this.isShowBigImg = false;
      },

      // 提交
      handlerSubmit() {
        const eljobId = this.eljobId;
        if (this.isSubmit) {
          return;
        }
        this.isSubmit = true;
        //console.log("submit", this.images);
        const resources = this.images.map(res => {
          return res.id;
        });
        const url = api.submitJob;
        this.$http.post(url, {
          eljobId,
          resources
        }).then(res => {
          this.$toast("提交成功");
          const num = history.length - this.len;
          console.log("important is", num);
          history.go(-num - 1 - 1);
        }).catch(ex => {
          this.isSubmit = false;
          console.error("提交失败", ex);
        })
      },

      // 取消提示
      handlerCancelTips() {
        this.showTips = false;
      },

      // 确定
      handlerSaveTips() {
        const num = history.length - this.len;
        //console.log("num is", num);
        this.showTips = false;
        history.go(-num - 1 - 1);
      },

    },
    beforeDestroy() {
      //console.log("beforeDestroy");
      //window.removeEventListener('popstate', this.goBack);
    },
    mounted() {
      //console.log("mounted ", this.images.length);
    },
    async created() {
      //history.go(1);
      const {hadSubmitNum, eljobId} = this.$route.query;
      this.allowNum = this.allowTotal - hadSubmitNum;
      this.eljobId = eljobId;

      if (this.allowNum < 0) {
        this.allowNum = 0;
      }

      //console.log("before history length is", history.length, this.images.length);
      history.pushState(null, null, document.URL);
      //console.log("after history length is", history.length, this.images.length);
      const vm = this;
      this.len = history.length;
      window.onpopstate = await function () {

        //console.log("hua hua");
        //console.log("====", vm.images, vm.images.length, document.URL);
        //console.log(vm.$route.meta.forbidBack, vm.images.length === 0, !vm.hadGone);

        if (vm.$route.meta.forbidBack && vm.images.length === 0 && !vm.hadGone) {
          vm.hadGone = true;
          //console.log("xixixi");
          history.back();
          return;
        }
        //console.log("iiiiii");
        if (vm.$route.meta.forbidBack && vm.images.length !== 0) {
          history.pushState(null, null, document.URL);
          //console.log("有图片的", history.length, vm.images.length);
          vm.showTips = true;
        }
      }
    }
  }
</script>

<style lang="less">
  .submit-photo {
    width: 100%;
    height: 100%;
    //border:1px solid red;

    // 标题
    .top-title {
      width: 100%;
      padding: 15px 24px;
      background: #FFEEC9;

      p {
        font-size: 28px;
        color: rgba(73, 80, 96, 1);
        line-height: 40px;
        span {
          font-weight: bold;
          color: #F25E5E;
        }
      }
    }

    // 图片
    .photo {
      margin-left: 24px;
      margin-top: 45px;
      //width:100%;
      //height:1024px;
      //border:1px solid red;
      //overflow: hidden;

      // 添加
      .photo-add {
        position: relative;
        display: inline-block;
        float: left;
        width: 140px;
        text-align: center;
        margin-right: 35px;
        // 添加图片
        .add-img {
          width: 100%;
          height: 140px;
          line-height: 120px;
          text-align: center;
          background: #D8D8D8;
          font-size: 100px;
          font-weight: bold;
          color: rgba(246, 246, 246, 1);
        }
        // 文本
        p {
          margin-top: 10px;
          font-size: 24px;
          color: rgba(73, 80, 96, 1);
          line-height: 33px;
        }
        // 实际上传
        .input-file {
          position: absolute;
          left: 0;
          top: 0;
          width: 140px;
          height: 183px;
          opacity: 0;
          z-index: 1;
          // border:1px solid red;
        }

      }

      // 照片
      .photo-img {
        position: relative;
        display: inline-block;
        float: left;
        margin-right: 35px;
        //margin-top: 40px;
        margin-bottom: 40px;
        width: 140px;
        //border:1px solid red;

        text-align: center;

        img {
          // border:1px solid green;
          width: 100%;
          height: 140px;
          object-fit: cover;
        }

        p {
          //margin-top:10px;
          font-size: 24px;
          color: rgba(73, 80, 96, 1);
          line-height: 33px;
        }

        // 删除
        .img-del {
          position: absolute;
          top: 0;
          right: 0;
          transform: translate(50%, -50%);
          width: 48px;
          height: 48px;
          svg {
            width: 100%;
            height: 100%;
          }
        }

      }
    }

    // 按钮
    .btn {
      position: absolute;
      bottom: 50px;
      left: 50%;
      transform: translate(-50%);

      button {
        width: 440px;
        height: 88px;
        background: rgba(23, 186, 255, 1);
        border-radius: 6px;
        font-size: 36px;
        color: rgba(255, 255, 255, 1);
        pointer-events: auto;
        border: none;
      }

      .disable {
        background: rgba(216, 220, 229, 1);
        pointer-events: none;
      }

    }

    .hidden {
      display: none !important;
    }
  }
</style>
