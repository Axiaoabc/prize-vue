<script>
import { onMounted, onUnmounted, ref, reactive, computed, toRefs } from 'vue';
import zhCN from 'ant-design-vue/es/locale/zh_CN';
import { message } from 'ant-design-vue';
import { allData } from './data';

export default {
  setup() {
    const particlesLoaded = async container => {
      console.log("Particles container loaded", container);
    };

    const state = reactive({
      prizeList: JSON.parse(localStorage.getItem('prizeList')) || [], // 后台配置的奖品数据
      prizeRecordList: [],
      isRunning: false, // 是否正在抽奖
      baseRunAngle: 360 * 5, // 总共转动角度 至少5圈
      prizeId: 0, // 中奖id
      editModalVisible: false, // 新增：编辑模态框可见性
      currentEditPrize: null, // 新增：当前编辑的奖品
    })
    const prizeWrap = ref(null)

    // 右侧中奖记录
    const rightHistory = ref(JSON.parse(localStorage.getItem('historyData')) || [])

    // 转盘奖品列表
    const dataSource = ref([])

    // 中奖记录columns
    const columns = [
      {
        title: "序号",
        dataIndex: 'index',
        align: 'center'
      },
      {
        title: '奖品名称',
        dataIndex: 'name',
        key: 'name',
      },
      {
        title: '奖品图片',
        dataIndex: 'img',
        key: 'img',
        scopedSlots: { customRender: 'img' }
      },
      {
        title: '操作',
        dataIndex: 'action',
        width: 80,
        align: 'center'
      }
    ]

    // 平均每个奖品角度
    const rotateAngle = computed(() => {
      const _degree = 360 / state.prizeList.length
      return _degree
    })

    // 要执行总角度数
    const totalRunAngle = computed(() => {
      return state.baseRunAngle + 360 - state.prizeId * rotateAngle.value - rotateAngle.value / 2
    })

    // 计算绘制转盘背景
    const bgColor = computed(() => {
      const _len = state.prizeList.length
      const colorList = ['#5352b3', '#363589', '#363596','#6452B3']
      let colorVal = ''
      for (let i = 0; i < _len; i++) {
        colorVal += `${colorList[i % colorList.length]} ${rotateAngle.value * i}deg ${rotateAngle.value * (i + 1)}deg,`
      }
      return `
            background: conic-gradient(${colorVal.slice(0, -1)});
          `
    })
    // 每个奖品布局
    const prizeStyle = computed(() => {
      const _degree = rotateAngle.value
      return (i) => {
        return `
              width: ${2 * 270 * Math.sin(_degree / 2 * Math.PI / 180)}px;
              height: 270px;
              transform: rotate(${_degree * i + _degree / 2}deg);
              transform-origin: 50% 100%;
            `
      }
    })

    onMounted(() => {
      prizeWrap.value.style = `${bgColor.value} transform: rotate(-${rotateAngle.value / 2}deg)`

      if (JSON.parse(localStorage.getItem('prizeList'))) {
        const prizeList = JSON.parse(localStorage.getItem('prizeList'))
        const keys = prizeList.map(item => item.key)
        settingState.selectedRowKeys = keys
      }

    })

    onUnmounted(() => {
      prizeWrap.value.removeEventListener('transitionend', stopRun)
    })

    // 获取随机数
    const getRandomNum = () => {
      // 过滤出可抽奖的奖品（数量大于0且概率大于0）
      const availablePrizes = state.prizeList.filter(prize => 
        prize.quantity > 0 && prize.percent > 0
      )

      if (availablePrizes.length === 0) {
        message.error('没有可抽取的奖品！')
        return -1
      }

      // 计算总概率
      const totalPercent = availablePrizes.reduce((sum, prize) => sum + prize.percent, 0)
      
      // 生成随机数 (0-100)
      const random = Math.random() * totalPercent
      
      // 根据概率权重选择奖品
      let currentSum = 0
      for (let i = 0; i < availablePrizes.length; i++) {
        currentSum += availablePrizes[i].percent
        if (random <= currentSum) {
          // 找到对应奖品在原始列表中的索引
          const prizeIndex = state.prizeList.findIndex(item => item.key === availablePrizes[i].key)
          return prizeIndex
        }
      }
      
      return 0 // 保底返回第一个奖品索引
    }

    const start = () => {
      if (state.prizeList.length > 0) {
        if (!state.isRunning) {
          state.isRunning = true

          console.log('开始抽奖，根据概率进行抽奖')
          const prizeId = getRandomNum()
          
          if (prizeId === -1) {
            state.isRunning = false
            return
          }

          state.prizeId = prizeId
          startRun()
        }
      } else {
        message.error('请添加奖品!');
      }
    }

    const handleparzeRecord = () => {
      console.log(state.prizeRecordList);
    }

    const startRun = () => {
      console.log(state.isRunning, totalRunAngle.value)
      // 设置动效
      prizeWrap.value.style = `
            ${bgColor.value}
            transform: rotate(${totalRunAngle.value}deg);
            transition: all 4s ease;
          `
      // 监听transition动效停止事件
      prizeWrap.value.addEventListener('transitionend', stopRun)
    }

    const stopRun = (e) => {
      console.log(e)
      state.isRunning = false
      prizeWrap.value.style = `
            ${bgColor.value}
            transform: rotate(${totalRunAngle.value - state.baseRunAngle}deg);
          `

      const winPrize = state.prizeList[state.prizeId]
      console.log('中奖ID>>>', state.prizeId, winPrize)
      
      // 减少奖品数量
      const prizeIndex = settingData.value.findIndex(item => item.key === winPrize.key)
      if (prizeIndex > -1) {
        settingData.value[prizeIndex].quantity -= 1
        // 同步更新 prizeList 中的数量
        state.prizeList[state.prizeId].quantity -= 1
      }

      rightHistory.value.push(winPrize)

      // 保存最新数据到本地存储
      localStorage.setItem('historyData', JSON.stringify(rightHistory.value))
      localStorage.setItem('prizeList', JSON.stringify(state.prizeList))
    }


    // 设置奖品modal
    const open = ref(false);
    const handleSettingModal = () => {
      open.value = true
    }
    const showModal = () => {
      open.value = true;
    };
    const handleOk = e => {
      console.log(e);
      if (settingState.selectedRowKeys.length > 0) {
        // 创建一个新数组，用来存放最终需要保留的数据
        let newPrizeList = [];

        settingState.selectedRowKeys.forEach(key => {
          // 从 settingData 中查找数据，而不是 allData
          const itemToAdd = settingData.value.find(item => item.key === key);
          if (itemToAdd) {
            newPrizeList.push(itemToAdd);
          }
        });

        // 更新 state.prizeList 为新数组，其中仅包含 settingState.selectedRowKeys 中存在的项
        state.prizeList = newPrizeList;
      } else {
        // 如果没有选中项，清空奖品列表
        state.prizeList = [];
      }

      console.log('最新数据:', settingState.selectedRowKeys, state.prizeList);
      localStorage.setItem('prizeList', JSON.stringify(state.prizeList))
      open.value = false;
    };

    const settingColumns = [
      {
        title: '序号',
        dataIndex: 'index',
        customRender: ({ index }) => index + 1
      },
      {
        title: '奖品名称',
        dataIndex: 'name',
      },
      {
        title: '奖品图片',
        dataIndex: 'img',
      },
      {
        title: '奖品数量',
        dataIndex: 'quantity',
      },
      {
        title: '奖品概率',
        dataIndex: 'percent',
      },
      {
        title: '操作',
        dataIndex: 'action',
        width: 100,
      }
    ];
    // const settingData = allData;
    const settingData = ref([...allData]);

    const settingState = reactive({
      selectedRowKeys: [],
      loading: false,
    });
    const hasSelected = computed(() => settingState.selectedRowKeys.length > 0);

    const onSelectChange = selectedRowKeys => {
      console.log('selectedRowKeys changed: ', selectedRowKeys);
      settingState.selectedRowKeys = selectedRowKeys;
    };

    // 分享
    const shareToCSDN = () => {
      const url = 'https://mp.csdn.net/mp_blog/creation/editor?spm=1011.2124.3001.6192';

      window.open(`${url}`);
    }

    // 新增：编辑模态框相关方法
    const showEditModal = (record = null) => {
      if (record) {
        // 编辑现有奖品
        state.currentEditPrize = { ...record }
      } else {
        // 新增奖品，不设置 key
        state.currentEditPrize = {
          name: '',
          pic: '',
          quantity: 1,
          percent: 0
        }
      }
      state.editModalVisible = true
    }

    const handleEditModalOk = () => {
      if (!state.currentEditPrize.name || !state.currentEditPrize.pic) {
        message.error('请填写完整信息！')
        return
      }

      if (state.currentEditPrize.key) {
        // 编辑现有奖品
        const index = settingData.value.findIndex(item => item.key === state.currentEditPrize.key)
        if (index > -1) {
          settingData.value[index] = { ...state.currentEditPrize }
        }
      } else {
        // 添加新奖品，在这里生成 key
        const newPrize = {
          ...state.currentEditPrize,
          key: Date.now().toString()
        }
        settingData.value.push(newPrize)
        // 自动选中新增的奖品
        settingState.selectedRowKeys.push(newPrize.key)
      }

      // 更新 prizeList
      const newPrizeList = settingData.value.filter(item =>
        settingState.selectedRowKeys.includes(item.key)
      )
      state.prizeList = newPrizeList

      // 保存到本地存储
      localStorage.setItem('prizeList', JSON.stringify(state.prizeList))

      state.editModalVisible = false
      message.success('保存成功！')
    }

    const locale = zhCN

    // 在 setup 函数中添加清空和删除历史记录的方法
    const clearHistory = () => {
      rightHistory.value = []
      localStorage.setItem('historyData', JSON.stringify([]))
      message.success('清空成功')
    }

    const deleteHistoryItem = (record, index) => {
      rightHistory.value.splice(index, 1)
      localStorage.setItem('historyData', JSON.stringify(rightHistory.value))
      message.success('删除成功')
    }

    return {
      ...toRefs(state),
      bgColor,
      prizeStyle,
      prizeWrap,
      start,
      handleparzeRecord,
      dataSource,
      columns,
      rightHistory,
      showModal,
      handleOk,
      handleSettingModal,
      open,
      hasSelected,
      onSelectChange,
      settingColumns,
      settingData,
      settingState,
      shareToCSDN,
      locale,
      particlesLoaded,
      showEditModal,
      handleEditModalOk,
      clearHistory,
      deleteHistoryItem,
    }
  }
}
</script>

<template>
  <a-config-provider :locale="locale">

    <div id="app" v-cloak>

      <div class="box">
        <!-- 转盘 -->
        <div class="container">
          <div class="prize-list" ref="prizeWrap" :style="bgColor">
            <div class="prize-item" v-for="(item, index) in prizeList" :style="prizeStyle(index)">
              <img :src="item.pic" alt="">
              <p>{{ item.name }}</p>
            </div>
          </div>

          <!-- 开始按钮 -->
          <div class="btn" @click="start"></div>
        </div>

        <!-- 设置奖品 -->
        <div class="settingPrizeBtn">
          <a-button type="primary" @click="handleSettingModal">设置奖品</a-button>
        </div>

        <!-- <div class="shareBtn">
          <a-button type="primary" @click="shareToCSDN">分享</a-button>
        </div> -->

        <!-- 中奖记录 -->
        <div class="histroy-list">
          <div style="margin-bottom: 16px; display: flex; justify-content: space-between; align-items: center;">
            <h3 style="margin: 0;">中奖记录</h3>
            <a-button type="primary" danger @click="clearHistory">
              清空记录
            </a-button>
          </div>
          <a-table :dataSource="rightHistory" :columns="columns" style="height: 500px;" size="small"
            :scroll="{ y: 400 }">
            <template #bodyCell="{ column, record, index }">
              <template v-if="column.dataIndex === 'index'">
                {{ index + 1 }}
              </template>

              <template v-if="column.dataIndex === 'img'">
                <img style="width: 50px; height: 50px;" :src="record.pic" alt="">
              </template>

              <template v-if="column.dataIndex === 'action'">
                <a-button type="link" danger @click="deleteHistoryItem(record, index)">
                  删除
                </a-button>
              </template>
            </template>
          </a-table>
        </div>
      </div>

    </div>


    <a-modal v-model:open="open" title="设置奖品" @ok="handleOk" width="1100px" cancelText="取消" okText="确定">
      <div style="margin-bottom: 16px">
        <a-button type="primary" @click="showEditModal()">新增奖品</a-button>
      </div>
      <a-table :row-selection="{ selectedRowKeys: settingState.selectedRowKeys, onChange: onSelectChange }"
        :scroll="{ y: 400 }" :columns="settingColumns" :data-source="settingData" style="width: 1050px !important;">
        <template #bodyCell="{ column, record, index }">
          <template v-if="column.dataIndex === 'img'">
            <img style="width: 50px; height: 50px;" :src="record.pic" alt="">
          </template>
          <template v-if="column.dataIndex === 'action'">
            <a @click="showEditModal(record)">编辑</a>
          </template>
        </template>
      </a-table>
    </a-modal>

    <!-- 新增：编辑奖品的模态框 -->
    <a-modal v-model:open="editModalVisible" :title="currentEditPrize?.key ? '编辑奖品' : '新增奖品'" @ok="handleEditModalOk"
      cancelText="取消" okText="确定">
      <a-form :model="currentEditPrize" layout="vertical">
        <a-form-item label="奖品名称" required>
          <a-input v-model:value="currentEditPrize.name" placeholder="请输入奖品名称" />
        </a-form-item>
        <a-form-item label="奖品图片" required>
          <a-input v-model:value="currentEditPrize.pic" placeholder="请输入图片URL" />
        </a-form-item>
        <a-form-item label="奖品数量">
          <a-input-number v-model:value="currentEditPrize.quantity" :min="1" />
        </a-form-item>
        <a-form-item label="中奖概率">
          <a-input-number v-model:value="currentEditPrize.percent" :min="0" :max="100" :formatter="value => `${value}%`"
            :parser="value => value.replace('%', '')" />
        </a-form-item>
      </a-form>
    </a-modal>

    <vue-particles id="tsparticles" @particles-loaded="particlesLoaded" url="http://foo.bar/particles.json" />

    <vue-particles id="tsparticles" @particles-loaded="particlesLoaded" :options="{
      background: {
        color: {
          value: '#000'
        }
      },
      fpsLimit: 120,
      interactivity: {
        events: {
          onClick: {
            enable: true,
            mode: 'push'
          },
          onHover: {
            enable: true,
            mode: 'repulse'
          },
        },
        modes: {
          bubble: {
            distance: 400,
            duration: 2,
            opacity: 0.8,
            size: 40
          },
          push: {
            quantity: 4
          },
          repulse: {
            distance: 200,
            duration: 0.4
          }
        }
      },
      particles: {
        color: {
          value: '#ffffff'
        },
        links: {
          color: '#ffffff',
          distance: 150,
          enable: true,
          opacity: 0.5,
          width: 1
        },
        move: {
          direction: 'none',
          enable: true,
          outModes: 'bounce',
          random: false,
          speed: 6,
          straight: false
        },
        number: {
          density: {
            enable: true,
          },
          value: 80
        },
        opacity: {
          value: 0.5
        },
        shape: {
          type: 'circle'
        },
        size: {
          value: { min: 1, max: 5 }
        }
      },
      detectRetina: true
    }" />
  </a-config-provider>
</template>

<style scoped lang="scss">
.settingPrizeBtn {
  position: relative;
  top: -330px;
  right: 730px;
}

.shareBtn {
  position: relative;
  top: -290px;
  right: 10px;
}

.box {
  position: relative;
  top: 100px;
  z-index: 2;
  margin-top: 100px;
  display: flex;
  align-items: center;

  .histroy-list {
    position: relative;
    left: -400px;
  }
}

[v-cloak] {
  display: none;
}

.container {
  left: -10px;
  width: 540px;
  height: 540px;
  /*background: #98d3fc url('https://www.jq22.com/demo/vue-luck-draw-pdmm202010260015/img/bg.a4b976d5.png') no-repeat center / 100% 100%;*/
  /*background: conic-gradient( 
       red 6deg, orange 6deg 18deg, yellow 18deg 45deg, 
       green 45deg 110deg, blue 110deg 200deg, purple 200deg);*/
  margin: 100px auto;
  position: relative;
}

.prize-list {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  border: 10px solid #98d3fc;
  overflow: hidden;
}

.prize-item {
  /*border: 2px solid red;*/
  position: absolute;
  left: 0;
  right: 0;
  top: -10px;
  margin: auto;
}

.prize-item img {
  width: 30%;
  height: 20%;
  margin: 40px auto 10px;
  display: block;
}

.prize-item p {
  color: #fff;
  font-size: 12px;
  text-align: center;
  line-height: 20px;
}

.btn {
  width: 160px;
  height: 160px;
  background: url('https://www.jq22.com/demo/jquerylocal201912122316/img/btn_lottery.png') no-repeat center / 100% 100%;
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  margin: auto;
  cursor: pointer;
}

.btn::before {
  content: "";
  width: 41px;
  height: 39px;
  background: url('https://www.jq22.com/demo/jquerylocal201912122316/img/icon_point.png') no-repeat center / 100% 100%;
  position: absolute;
  left: 0;
  right: 0;
  top: -33px;
  margin: auto;
}

:deep(.ant-table) {
  width: 400px;
}
</style>
