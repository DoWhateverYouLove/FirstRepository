<template>
  <x-animate :config="config">
    <div slot="animate-box" v-show="config.show">
      <div class="x-time-picker" @touchmove.prevent="">
        <div class="x-time-picker-wrap">
          <!--顶部菜单-->
          <div class="time-picker-title flex-between color-deepgray">
            <div class="cancel" @click="cancelPicker()">取消</div>
            <div class="confirm" @click="selectTime()">确定</div>
          </div>
          <!--底部内容-->
          <div :class="`select-content`">
            <div class="line" id="selected_line"></div>
            <template v-for="(item, level) in timeData">
              <div class="select-item" :key="level" :data-level="level" :data-length="item.length"
                   v-picker="{ selectData }" v-show="item.length > 0">
                <div class="select-item-wrap">
                  <div class="time-item" v-for="value in item" :data-value="value.value" :key="value.value"
                       :class="selectResult[level] && selectResult[level].name === value.name ? '' : 'Z15'">
                    {{ value.name }}
                  </div>
                </div>
              </div>
            </template>
          </div>
        </div>
      </div>
    </div>
  </x-animate>

</template>

<script>
  export default {
    name: "ui-time-picker",
    props: [],
    data() {
      return {
        itemHeight: 0, // 可选中行高度
        itemElem: [], // 可选择的行元素
        selectResult: [], // 选择结果
        config: {}, // 配置信息
        oldCenterData: [], // 备份的中数据组
        oldRightData: [], // 备份的右数据组
        timeData: [], // 总体数据
        info: {}, // 同步事件处理
        cityInfo: [], // 城市类型数据组存储
        cityLoadSuccess: false, // 城市数据加载是否已完成
        promise: '' // 同步事件注册
      }
    },
    watch: {
      /** 已选项样式初始化 && 数据重置 */
      'config.show'(val) {
        if (val) {
          setTimeout(() => {
            this.getItemElem(); // 获取元素
            this.timeData.forEach((item, index) => {
              this.scrollTo(index, 0);
            });
          }, 500);
        }
      }
    },
    /** 全局指令注册 */
    directives: {
      picker: {
        bind(el, binding, vNode) {
          let startY = 0; // 开始位置
          let differY = 0; // 滑动位置距离差
          let translateY = 0; // Y轴偏移值
          let itemLen = 0; // 列数量
          let wrapEle = el.childNodes[0]; // 外部容器
          let itemHeight = 0; // 被选中的元素样式高度
          let level; // 滑动层级
          /** 滑动开始 */
          el.addEventListener('touchstart', (e) => {
            level = Number(el.getAttribute('data-level')); // 获取滑动层级
            startY = e.touches[0].clientY; // 记录Y轴起始位置
            itemLen = Number(el.getAttribute('data-length')); // 获取选项数量长度
            if (el.childNodes[0].style.transform) { // 初始化Y轴偏移值
              translateY = Number(el.childNodes[0].style.transform.replace(/[^\d|-]/g, ''));
            }
          });
          /** 滑动中 */
          el.addEventListener('touchmove', (e) => {
            differY = e.touches[0].clientY - startY; // Y轴滑动距离
            wrapEle.style.transition = ``; // 外部容器卸载缓动样式
            wrapEle.style.transform = `translateY(${translateY + differY}px)`; // 容器Y轴样式变化
          });
          /** 滑动结束 */
          el.addEventListener('touchend', () => {
            let scrollToIndex = 0;
            translateY = differY + translateY;
            wrapEle.style.transition = `all .3s`; // 外部容器挂载缓动样式
            /** 样式黏着效果 */
            if (translateY > 0) { // 超过顶部范围
              translateY = 0;
              wrapEle.style.transform = `translateY(${translateY}px)`; // 容器Y轴样式变化
            } else { // 超过底部范围
              itemHeight = document.getElementById('selected_line').offsetHeight;
              scrollToIndex = Math.round(Math.abs(translateY) / itemHeight);
              if (scrollToIndex >= itemLen) {
                scrollToIndex = itemLen - 1;
              }
              translateY = -scrollToIndex * (itemHeight); // 滑动至选择项顶部位置
              wrapEle.style.transform = `translateY(${translateY}px)`;
            }
            binding.value.selectData(scrollToIndex, level); // 写入选择数据
          });
        }
      }
    },
    /** 页面新建 */
    created() {
      let _this = this;
      Global.timePicker = function (obj) {
        return _this.openTimePicker(obj);
      };
      /*获取省市区数据*/
      $.get('../static/json/addressV0.json', (res) => {
        this.cityInfo = res;
      });
    },
    methods: {
      /** 打开时间选择器 */
      openTimePicker(obj) {
        this.config = {
          type: obj.type ? obj.type : 'slide',
          direction: obj.direction ? obj.direction : 'bottom',
          show: obj.show ? obj.show : false,
          noCancelButton: obj.noCancelButton ? obj.noCancelButton : true,
          leftData: obj.leftData ? obj.leftData : [],
          centerData: obj.centerData ? obj.centerData : [],
          rightData: obj.rightData ? obj.rightData : [],
          dataType: obj.dataType ? obj.dataType : ''
        };
        this.cityLoadSuccess = false;
        /** 针对不同类型的数据进行首列初始化处理 */
        switch (this.config.dataType) {
          /** 时间 */
          case 'time':
            /** 还原默认选项 */
            this.config.leftData = [];
            this.config.centerData = [];
            this.config.rightData = [];
            /** 时 */
            for (let i = 0; i < 24; i++) {
              this.config.leftData.push({
                name: i < 10 ? '0' + i : String(i),
                value: i < 10 ? '0' + i : String(i)
              });
            }
            /** 分、秒 */
            for (let i = 0; i < 60; i++) {
              this.config.centerData.push({
                name: i < 10 ? '0' + i : String(i),
                value: i < 10 ? '0' + i : String(i)
              });
              this.config.rightData.push({
                name: i < 10 ? '0' + i : String(i),
                value: i < 10 ? '0' + i : String(i)
              });
            }
            this.timeData = [this.config.leftData, this.config.centerData, this.config.rightData];
            break;
          /** 省市区 */
          case 'city':
            /** 还原默认选项 */
            this.config.leftData = [];
            this.config.centerData = [];
            this.config.rightData = [];
            /** 省市区数据初始化 */
            this.cityInfo.map((item) => {
              this.config.leftData.push({
                name: item.enCityName,
                value: item.enCityCode
              });
            });
            this.cityLoadSuccess = true; // 城市数据加载完毕
            break;
        }
        this.timeData = [this.config.leftData, this.config.centerData, this.config.rightData];
        this.promise = new Promise((res, rej) => {
          this.info.res = res;
          this.info.rej = rej;
        });
        return this.promise;
      },
      /** 关闭时间选择器 */
      closeTimePicker() {
        this.config.show = false;
      },
      /** 取消选择时间 */
      cancelPicker() {
        this.info.rej(this.selectResult);
        this.closeTimePicker();
      },
      /** 选择数据，数据外传 */
      selectTime() {
        let tempData;
        switch (this.config.dataType) {
          /** 用车时间 */
          case 'taxi':
            let currentDay = new Date().timeFormat('YYYY-MM-DD'); // 当前日期
            let currentTime = new Date().timeFormat('hh:mm:ss'); // 当前时间
            let tempDate = this.selectResult[0].value ? this.selectResult[0].value : currentDay;
            let tempTime = this.selectResult[1].name ? this.selectResult[1].name + ':' + this.selectResult[2].name + ':' + '00' : currentTime;
            tempData = {
              type: this.selectResult[0].name === '现在' ? 0 : 1,
              day: this.selectResult[0].name,
              departureTime: new Date(tempDate + ' ' + tempTime).timeFormat('YYYY-MM-DD hh:mm:ss')
            }
            break;
          /** 日期 */
          case 'date':
            tempData = this.selectResult[0].value + '-' + this.selectResult[1].value + '-' + this.selectResult[2].value;
            break;
          /** 时间 */
          case 'time':
            tempData = this.selectResult[0].value + ':' + this.selectResult[1].value + ':' + this.selectResult[2].value;
            break;
          /** 省市区 */
          case 'city':
            tempData = {
              province: this.selectResult[0].name || '',
              provinceCode: this.selectResult[0].value || '',
              city: this.selectResult[1].name || '',
              cityCode: this.selectResult[1].value || '',
              district: this.selectResult[2].name || '',
              districtCode: this.selectResult[2].value || ''
            };
            // tempData = this.selectResult[0].value + this.selectResult[1].value + this.selectResult[2].value;
            break;
          /** 默认 */
          default:
            tempData = {
              leftData: this.selectResult[0].value,
              centerData: this.selectResult[1].value,
              rightData: this.selectResult[2].value
            };
            break;
        }
        this.info.res(tempData);
        this.closeTimePicker();
      },
      /** 选择项重置 */
      scrollTo(level, index) {
        let translateY = -index * this.itemHeight;
        this.itemElem[level].childNodes[0].style.transform = `translateY(${translateY}px)`;
        this.selectData(index, level); // 数据写入
      },
      /** 获取选择器对象 */
      getItemElem() {
        this.itemElem = document.querySelectorAll('.select-item');
      },
      /** 重置选项位置 */
      resetTransform(index) {
        Array.from(this.itemElem).forEach((item, i) => {
          if (i > index) {
            item.childNodes[0].style.transform = '';
          }
        })
      },
      /* 处理城市信息 */
      handlerCityInfo(index, level) {
        let i, o, secondData, thirdData = []; // 第二级联和第三级联数据声明
        /*市数据取出*/
        for (i = 0; i < this.cityInfo.length; i++) {
          if (this.selectResult[0].value === this.cityInfo[i].enCityCode) { // 一级选中数据对比
            secondData = this.cityInfo[i].resultList ? this.cityInfo[i].resultList : [];
            break;
          }
        }
        secondData.forEach((item) => {
          this.config.centerData.push({
            name: item.enCityName,
            value: item.enCityCode
          });
        });
        this.timeData[1] = this.config.centerData;
        if (level === 0) {
          this.$set(this.selectResult, 1, this.config.centerData[0]);
        }
        /*县级数据写入*/
        for (o = 0; o < secondData.length; o++) {
          if (this.selectResult[1].value === secondData[o].enCityCode) {
            thirdData = secondData[o].resultList ? secondData[o].resultList : [];
            break;
          }
        }
        thirdData.forEach((item) => {
          this.config.rightData.push({
            name: item.enCityName,
            value: item.enCityCode
          });
        });
        this.timeData[2] = this.config.rightData;
        this.$set(this.selectResult, 2, this.config.rightData[0]);
      },
      /** 滑动数据初始化 && 滑动结束后回调，选择数据 */
      selectData(index, level) {
        this.resetTransform(level); // 重置选项位置
        this.$set(this.selectResult, level, this.timeData[level][index]); // 选中数据写入（数据判断处理前，更新）
        /** 针对滑动数据进行初始化处理 */
        switch (this.config.dataType) {
          /** 用车 */
          case 'taxi':
            let tempHour = new Date().getHours(); // 现在时
            let tempMinute = new Date().getMinutes(); // 现在分
            let hourNum = 0; // 小时值
            let minuteNum = 0; // 分钟值
            /** 还原默认选项 */
            this.timeData[1] = [];
            this.timeData[2] = [];
            /** 初始化默认可选项 */
            if (this.selectResult[0].name !== '现在') {
              switch (this.selectResult[0].name) {
                case '今天':
                  hourNum = (tempMinute + 40) >= 60 ? (tempHour + 1) : tempHour;
                  if ((tempMinute + 40) >= 60) { // 预约40分钟后超过1小时
                    minuteNum = Math.ceil(((tempMinute + 40) - 60) / 10);
                  } else { // 预约40分钟后不超过1小时
                    if (Math.ceil(tempMinute / 10) + 4 < 6) { // 预约40分钟后不等于1小时
                      if (this.selectResult[1] && this.selectResult[1].value - tempHour === 1) { // 选择时间与当前时间计算等于1小时
                        minuteNum = 0; // 下个小时分钟重置
                      } else {
                        minuteNum = Math.ceil((tempMinute + 40) / 10);
                      }
                    } else { // 预约40分钟后等于1小时
                      minuteNum = 5;
                    }
                  }
                  break;
                default:
                  hourNum = 0;
                  minuteNum = 0;
                  break;
              }
              this.getHour(hourNum);
              this.getMinute(minuteNum);
            }
            /** 重置默认数据 */
            this.timeData = [this.config.leftData, this.config.centerData, this.config.rightData];
            /** 写入默认选中数据 */
            if (this.selectResult[0].name !== '现在') {
              if (level === 0) { // 写入第二级默认选中数据
                this.$set(this.selectResult, 1, this.timeData[1][0]);
                this.$set(this.selectResult, 2, this.timeData[2][0]);
              }
            } else {
              this.$set(this.selectResult, 1, []);
              this.$set(this.selectResult, 2, []);
              this.timeData[1] = [];
              this.timeData[2] = [];
            }
            /** 初始化三级可选项 */
            if (this.selectResult[1] && this.selectResult[1].value - tempHour > 1) { // 选择时间与当前时间计算大于1小时
              minuteNum = 0;
              this.getMinute(minuteNum);
              if (level === 1) { // 写入第三级默认选中数据
                this.$set(this.selectResult, 2, this.timeData[2][0]);
              }
            }
            break;
          /** 城市 */
          case 'city':
            this.config.centerData = [];
            this.config.rightData = [];
            this.handlerCityInfo(index, level);
            break;
        }
        this.$set(this.selectResult, level, this.timeData[level][index]); // 选中数据写入（数据判断处理后，写入）
      },
      /** 取小时值 */
      getHour(time) {
        let tempObj;
        this.config.centerData = [];
        for (let i = time; i <= 23; i++) {
          tempObj = {
            name: i < 10 ? ('0' + i) : i.toString(),
            value: i
          };
          this.config.centerData.push(tempObj);
        }
        this.timeData = '';
        this.timeData = [this.config.leftData, this.config.centerData, this.config.rightData];
      },
      /** 取分钟值 */
      getMinute(time) {
        let tempObj;
        this.config.rightData = [];
        for (let i = time; i <= 5; i++) {
          tempObj = {
            name: i + '0',
            value: Number(i + '0')
          };
          this.config.rightData.push(tempObj);
          this.timeData = '';
          this.timeData = [this.config.leftData, this.config.centerData, this.config.rightData];
          // this.timeData[2] = this.config.rightData;
        }
      }
    }
  }
</script>

<style lang="scss" scoped>
  .x-time-picker {
    .x-time-picker-wrap {
      height: 430px;
      background-color: #fff;
      .time-picker-title {
        padding: 30px;
        font-size: 28px;
      }
      .select-content {
        position: relative;
        height: 300px;
        text-align: center;
        .select-item {
          float: left;
          text-align: center;
          overflow: hidden;
          width: 33.33%;
          height: 100%;
          font-size: 36px;
          transition: width .3s;
        }
        .select-item-wrap {
          position: relative;
          z-index: 2;
          padding: 70px 15px 0;
        }
        .time-item {
          height: 70px;
          white-space: nowrap;
          text-overflow: ellipsis;
          overflow: hidden;
        }
        .line {
          position: absolute;
          width: 100%;
          height: 70px;
          top: 70px;
          border-top: 2px #f2f2f2 solid;
          border-bottom: 2px #f2f2f2 solid;
          box-sizing: border-box;
        }
      }
    }
  }
</style>
