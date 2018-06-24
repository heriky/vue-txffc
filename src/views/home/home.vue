<template>
    <div class="root">
        <div class="h-group" style="justify-content: flex-start">
            <span>数据刷新倒计时:</span>
            <span>&nbsp;&nbsp;{{59 - second}}</span>
        </div>

        <p>最近10次开奖号码(千百十个)：</p>
        <div>{{isLoading ? '正在刷新...' : ''}}</div>
        <ul class="recent-list">
            <li v-for="(it, index) in recentNum" :key="index"
                class="prev-prize"
                v-if="index ===0 || isExpanded">
                <div class="h-group" style="justify-content: flex-start">
                    <span @click="toggle(index)" class="prize-item">{{it}}</span>
                    <img v-if="index === 0"
                         src="static/fold.png"
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
            </tr>
            <tr v-for="(item, index) in configs" :key="index"
                :style="{color: item.count >= 3 ? 'red' : 'currentColor', fontWeight: item.count >= 3 ? 'bold' : 0}">
                <td>{{item.label}}</td>
                <td>{{item.count}}</td>
            </tr>
        </table>
    </div>
</template>

<script>
  export default {
    name: "home",
    data() {
      return {
        rs: '00000',
        rsList: [],
        second: 0,
        isExpanded: false,
        isLoading: false,
        configs: [{
          label: '千百',
          pos: [0, 1],
          count: 0
        }, {
          label: '千十',
          pos: [0, 2],
          count: 0
        }, {
          label: '千个',
          pos: [0, 3],
          count: 0
        }, {
          label: '百十',
          pos: [1, 2],
          count: 0
        }, {
          label: '百个',
          pos: [1, 3],
          count: 0
        }, {
          label: '十个',
          pos: [2, 3],
          count: 0
        }]
      };
    },
    computed: {
      recentNum() {
        return this.rsList.map(item => {
          return item['onlinenumber'].toString().substr(-4);;
        })
      }
    },
    methods: {
      toggle(index) {
        if(index !== 0) return;
        this.isExpanded = !this.isExpanded;
      },
      handleData(rs) {
        this.configs.forEach(config => {
          const i = config.pos[0];
          const j = config.pos[1];
          const change = Math.abs(rs[i] - rs[j]);

          // 不相邻且不相连的情况下，就置为0
          // 相邻或者相连，则加1
          config.count = ( change === 1 || change === 0 || change === 9 ) ? config.count + 1 : 0;
        });
      },
      tick() {
        // 每次新的一轮开始时，前十秒刷新数据
        const second = new Date().getSeconds();
        this.second = second;
        if (second < 10) {
          this.fetchResult().then(() => {
            this.timer = setTimeout(this.tick, 1000);
          }).catch(err => {
            alert(err || '获取数据出错');
          })
        } else {
          this.timer = setTimeout(this.tick, 1000);
        }
      },
      fetchResult() {
        if(this.isLoading) { return Promise.resolve(false)}
        this.isLoading = true;
        return this.$http.get('/api/tencent/onlineim').then(res => {
          this.rsList = res;

          const dataStr = res[0]['onlinenumber'];
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
</style>
