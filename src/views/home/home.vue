<template>
    <div class="root">
        <img src="../../assets/avatar.jpg" class="avatar" :class="{rotated: inFetching}">
        <div>{{inFetching || isLoading? '正在刷新...' : '等待刷新'}}</div>

      <hr>

      <div class="h-group" style="justify-content: flex-start">
            <span>数据刷新倒计时:</span>
            <span>&nbsp;&nbsp;{{59 - second}}</span>
        </div>

        <p style="text-align: left">最近几次次开奖号码(千百十个)：</p>
        <ul class="recent-list">
            <li v-for="(it, index) in recentNum" :key="index"
                class="prev-prize"
                v-if="index ===0 || isExpanded">
                <div class="h-group" style="justify-content: flex-start">
                    <span @click="toggle(index)" class="prize-item">{{it}}</span>
                    <img v-if="index === 0"
                         :src="fold"
                         @click="toggle(index)"
                         alt="" class="fold-icon" :class="{unfold: isExpanded}">
                </div>
            </li>
        </ul>

        <!--组合次数-->
        <table class="table">
            <tr>
                <th>组合</th>
                <th>次数</th>
                <th title="计算方式：累计未中奖的次数 / 总体轮次">整体中奖率 ?</th>
            </tr>
            <tr v-for="(item, index) in configs" :key="index"
                :style="{color: item.count >= 3 ? 'red' : 'currentColor', fontWeight: item.count >= 3 ? 'bold' : 0}">
                <td>{{item.label}}</td>
                <td>{{item.count}}</td>
                <td>{{(item.history / totalCount).toFixed(2)}}</td>
            </tr>
        </table>
    </div>
</template>

<script>
  import fold from './assets/fold.png';
  export default {
    name: "home",
    data() {
      return {
        inFetching: false, // 是否在刷新时间区域内
        totalCount: 1,
        fold,
        rs: '00000',
        rsList: [],
        second: 0,
        isExpanded: false,
        isLoading: false,
        configs: [{
          label: '千百',
          pos: [0, 1],
          count: 0,
          history: 0,
        }, {
          label: '千十',
          pos: [0, 2],
          count: 0,
          history: 0,
        }, {
          label: '千个',
          pos: [0, 3],
          count: 0,
          history: 0,
        }, {
          label: '百十',
          pos: [1, 2],
          count: 0,
          history: 0,
        }, {
          label: '百个',
          pos: [1, 3],
          count: 0,
          history: 0,
        }, {
          label: '十个',
          pos: [2, 3],
          count: 0,
          history: 0,
        }]
      };
    },
    computed: {
      recentNum() {
        return this.rsList.map((item, index) => {
          return item['code'].toString().substr(-4);
        })
      }
    },
    methods: {
      toggle(index) {
        if(index !== 0) return;
        this.isExpanded = !this.isExpanded;
      },
      handleData(rs) {
        this.totalCount += 1; //总的开奖册数
        this.configs.forEach(config => {
          const i = config.pos[0];
          const j = config.pos[1];
          const change = Math.abs(rs[i] - rs[j]);

          // 不相邻且不相连的情况下，就置为0
          // 相邻或者相连，则加1
          config.count = ( change === 1 || change === 0 || change === 9 ) ? config.count + 1 : 0;
          config.history += ( change === 1 || change === 0 || change === 9 ) ? 1 : 0;
        });
      },
      tick() {
        // 每次新的一轮开始时，前十秒刷新数据
        if (this.second < 15) {
          this.inFetching = true;
          this.fetchResult().then(() => {
            this.timer = setTimeout(this.tick, 1000);
          }).catch(err => {
            alert('获取数据出错, 请务必刷新页面');
            clearTimeout(this.timer);
            this.timer = null;
          })
        } else {
          this.inFetching = false;
          this.timer = setTimeout(this.tick, 1000);
        }
      },
      fetchResult() {
        if(this.isLoading) { return Promise.resolve(false)}
        this.isLoading = true;
        return this.$http.get('/api/tencent/onlineim').then(res => {
          this.rsList = res.data.filter(item => item.time.endsWith('00'));

          const dataStr = this.rsList[0]['code'];
          let rs = dataStr.toString().substr(-4);

          // 当数据有变化的时候才进行处理, 此时清除定时器
          if (rs !== this.rs) {
            this.handleData(rs);
            this.rs = rs;
          }

          this.isLoading = false;
          return this.rs;
        });
      }
    },
    mounted() {
      // 第一次拉取数据
      this.fetchResult();
      if (this.timer) {
        clearTimeout(this.timer);
        this.timer = null;
      }
      this.timer = setTimeout(this.tick, 1000);

      // 倒计时，不停歇脚
      if (this.countTimer) {
        clearTimeout(this.countTimer);
        this.countTimer = null;
      }
      this.countTimer = setInterval(() => {
        this.second = new Date().getSeconds();
      }, 1000);

    }
  }
</script>

<style scoped>
    .root {
        padding: 10px;
        font-family: Microsoft YaHei, sans-serif;
    }
    .recent-list {
        list-style: none;
        font-size: 24px;
        padding: 0;
    }

    .prev-prize {
        letter-spacing: 10px;
    }
    .prev-prize:first-child {
        text-decoration: underline;
        font-weight: bold;
    }
    .toggle {
        cursor: pointer;
    }
    .table {
        border-spacing: 0;
        border-collapse: collapse;
        width: 80%;
        margin: 0 auto;
    }
    .table tr:nth-child(even) {
        background-color: #eee;
    }
    .table td {
        text-align: center;
        border: 1px solid #cecece;
        padding: 5px 10px;
    }

    .h-group {
        display: flex;
        flex-flow: row nowrap;
        align-items: center;
        justify-content: center;
    }

    .v-group {
        display: flex;
        flex-flow: column nowrap;
        align-items: center;
        justify-content: center;
    }
    .fold-icon {
        cursor: pointer;
        width: 14px;
        height: 14px;
        transform: rotate(180deg);
    }
    .fold-icon.unfold {
        transform: rotate(0);
    }

    .prize-item {
        cursor: pointer;
    }
    .table th {
      border: 1px solid #cecece;
      background: #c3d9ff;
      padding: 10px;
    }
    .avatar {
      width: 200px;
      height: 200px;
      border-radius: 50%;
    }
    .avatar.rotated {
      -webkit-animation: r 2s ease-in-out infinite;
      -o-animation: r 2s ease-in-out infinite;
      animation: r 2s ease-in-out infinite;
    }
    @-webkit-keyframes r {
      from { transform: rotate(0deg) }
      to { transform: rotate(360deg) }
    }
  @keyframes r {
    from { transform: rotate(0deg) }
    to { transform: rotate(360deg) }
  }
</style>
