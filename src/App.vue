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
      historyModalVisible: false, // 新增：中奖记录模态框可见性
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
      const colorList = ['#FCF2D7', '#F8E2AF', '#E6C98F', '#F5E8C6']
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

      // 加载保存的数据
      if (JSON.parse(localStorage.getItem('prizeList'))) {
        const prizeList = JSON.parse(localStorage.getItem('prizeList'))
        const keys = prizeList.map(item => item.key)
        settingState.selectedRowKeys = keys

        // 如果本地存储中有 settingData，使用它更新 settingData
        const savedSettingData = localStorage.getItem('settingData')
        if (savedSettingData) {
          settingData.value = JSON.parse(savedSettingData)
        }
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

      // 生成随机数 (0-总概率)
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

      return -1
    }

    const start = () => {
      if (state.prizeList.length > 0) {
        if (!state.isRunning) {
          // 检查是否还有可抽奖品
          const availablePrizes = state.prizeList.filter(prize => prize.quantity > 0)
          if (availablePrizes.length === 0) {
            message.error('所有奖品已抽完！')
            return
          }

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
        message.error('请添加奖品!')
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

      // 检查奖品是否还有库存
      if (winPrize.quantity <= 0) {
        message.error('该奖品已无库存！')
        return
      }

      // 减少奖品数量
      const prizeIndex = settingData.value.findIndex(item => item.key === winPrize.key)
      if (prizeIndex > -1) {
        // 更新 settingData 中的数量
        settingData.value[prizeIndex].quantity -= 1

        // 更新 prizeList 中的数量
        const prizeListIndex = state.prizeList.findIndex(item => item.key === winPrize.key)
        if (prizeListIndex > -1) {
          state.prizeList[prizeListIndex].quantity -= 1
        }

        // 添加到中奖记录
        rightHistory.value.push({
          ...winPrize,
          quantity: state.prizeList[prizeListIndex].quantity // 记录剩余数量
        })

        // 保存所有更新
        saveSettingData()
        localStorage.setItem('prizeList', JSON.stringify(state.prizeList))
        localStorage.setItem('historyData', JSON.stringify(rightHistory.value))
      }
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
      console.log('最新数据settingData', settingData.value);
      localStorage.setItem('prizeList', JSON.stringify(state.prizeList))
      localStorage.setItem('initialSettingData', JSON.stringify(settingData.value))
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
        width: 160,
        align: 'center'
      }
    ];
    // const settingData = allData;
    const settingData = ref(JSON.parse(localStorage.getItem('settingData')) || [...allData]);

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
      saveSettingData() // 保存 settingData

      state.editModalVisible = false
      message.success('保存成功！')
    }

    const locale = zhCN

    // 修改清空历史记录的方法
    const clearHistory = () => {
      // 清空历史记录
      rightHistory.value = []
      localStorage.setItem('historyData', JSON.stringify([]))

      // 重置所有奖品的数量为初始值
      const savedSettingData = localStorage.getItem('settingData')
      const initialSettingData = JSON.parse(localStorage.getItem('initialSettingData')) || [...allData]

      settingData.value.forEach((item, index) => {
        // 找到对应的初始数据
        const initialItem = initialSettingData.find(initial => initial.key === item.key)
        if (initialItem) {
          item.quantity = initialItem.quantity
        }
      })
      console.log('clearHistory', settingData.value)

      // 同步更新 prizeList 中的数量
      state.prizeList.forEach(prize => {
        const settingItem = settingData.value.find(item => item.key === prize.key)
        if (settingItem) {
          prize.quantity = settingItem.quantity
        }
      })

      // 保存更新后的数据
      saveSettingData()
      localStorage.setItem('prizeList', JSON.stringify(state.prizeList))

      message.success('清空成功')
    }

    const deleteHistoryItem = (record, index) => {
      rightHistory.value.splice(index, 1)
      localStorage.setItem('historyData', JSON.stringify(rightHistory.value))
      message.success('删除成功')
    }

    // 添加保存 settingData 的方法
    const saveSettingData = () => {
      localStorage.setItem('settingData', JSON.stringify(settingData.value))
    }

    // 添加删除奖品的方法
    const deletePrize = (record) => {
      // 从 settingData 中删除
      const index = settingData.value.findIndex(item => item.key === record.key)
      if (index > -1) {
        settingData.value.splice(index, 1)
      }

      // 从选中项中移除
      const selectedIndex = settingState.selectedRowKeys.indexOf(record.key)
      if (selectedIndex > -1) {
        settingState.selectedRowKeys.splice(selectedIndex, 1)
      }

      // 从 prizeList 中移除
      state.prizeList = state.prizeList.filter(item => item.key !== record.key)

      // 保存更新
      saveSettingData()
      localStorage.setItem('prizeList', JSON.stringify(state.prizeList))
      message.success('删除成功')
    }

    // 添加打开中奖记录的方法
    const showHistoryModal = () => {
      state.historyModalVisible = true
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
      saveSettingData,
      deletePrize,
      showHistoryModal,
    }
  }
}
</script>

<template>
  <a-config-provider :locale="locale">

    <div id="wrapper" v-cloak >

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
          <a-space>
            <a-button type="primary" @click="handleSettingModal">设置奖品</a-button>
            <a-button type="primary" @click="showHistoryModal">中奖记录</a-button>
          </a-space>
        </div>

        <!-- <div class="shareBtn">
          <a-button type="primary" @click="shareToCSDN">分享</a-button>
        </div> -->

        <!-- 中奖记录 -->
        <a-modal
          v-model:open="historyModalVisible"
          title="中奖记录"
          width="800px"
          :footer="null"
          :style="{ top: '10%' }"
          wrapClassName="history-modal"
        >
          <div style="margin-bottom: 16px; display: flex; justify-content: flex-end;">
            <a-button type="primary" danger @click="clearHistory" :disabled="!rightHistory.length">
              清空记录
            </a-button>
          </div>
          <a-table 
            :dataSource="rightHistory" 
            :columns="columns" 
            size="small"
            :scroll="{ y: 400 }"
            style="width: 100% !important;"
          >
            <template #bodyCell="{ column, record, index }">
              <template v-if="column.dataIndex === 'index'">
                {{ index + 1 }}
              </template>

              <template v-if="column.dataIndex === 'img'">
                <img style="width: 50px; height: 50px;" :src="record.pic" alt="">
              </template>
            </template>
          </a-table>
        </a-modal>
      </div>

    </div>


    <a-modal 
      v-model:open="open" 
      title="设置奖品" 
      @ok="handleOk" 
      width="900px" 
      cancelText="取消" 
      okText="确定"
      :style="{ top: '10%' }"
      wrapClassName="setting-modal"
    >
      <div style="margin-bottom: 16px">
        <a-button type="primary" @click="showEditModal()">新增奖品</a-button>
      </div>
      <a-table :row-selection="{ selectedRowKeys: settingState.selectedRowKeys, onChange: onSelectChange }"
        :scroll="{ y: 400 }" :columns="settingColumns" :data-source="settingData" style="width: 100% !important;">
        <template #bodyCell="{ column, record, index }">
          <template v-if="column.dataIndex === 'img'">
            <img style="width: 50px; height: 50px;" :src="record.pic" alt="">
          </template>
          <template v-if="column.dataIndex === 'action'">
            <a-space>
              <a @click="showEditModal(record)">编辑</a>
              <a-divider type="vertical" />
              <a-popconfirm title="确定要删除这个奖品吗？" @confirm="deletePrize(record)" ok-text="确定" cancel-text="取消">
                <a style="color: #ff4d4f">删除</a>
              </a-popconfirm>
            </a-space>
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

    <!-- <vue-particles id="tsparticles" @particles-loaded="particlesLoaded" url="http://foo.bar/particles.json" /> -->

    <!-- <vue-particles id="tsparticles" @particles-loaded="particlesLoaded" :options="{
      background: {
        color: {
          value: ''
        },
        image:'/bg_prizes.jpg'
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
    }" /> -->
  </a-config-provider>
</template>

<style scoped lang="scss">
// body {
//   background-image: url(/bg_prizes.jpg);
// }

.settingPrizeBtn {
  position: absolute;
  bottom: 5%;
  right: 4%;
  
  // 确保按钮之间有间距
  :deep(.ant-space) {
    gap: 8px !important;
  }
}

.shareBtn {
  position: relative;
  top: -290px;
  right: 10px;
}

.box {
  position: relative;
  z-index: 2;
  // margin-top: 100px;
  display: flex;
  align-items: center;
  width: 100%;
  height: 100%;
}

[v-cloak] {
  display: none;
}
#wrapper{
  width: 100vw;
  height: 100vh;
  background-image: url(/bg_prizes.jpg);
  background-repeat: no-repeat;
  background-size: cover;
  background-position: top center;
}
.container {
  left: -10px;
  width: 540px;
  height: 540px;
  /*background: #98d3fc url('https://www.jq22.com/demo/vue-luck-draw-pdmm202010260015/img/bg.a4b976d5.png') no-repeat center / 100% 100%;*/
  /*background: conic-gradient( 
       red 6deg, orange 6deg 18deg, yellow 18deg 45deg, 
       green 45deg 110deg, blue 110deg 200deg, purple 200deg);*/
  margin: 16% auto 0;
  position: relative;
}

.prize-list {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  border: 10px solid #fff;
  outline: 1px solid #000;
  overflow: hidden;
  position: relative;
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
  width: 20%;
  height: 20%;
  margin: 40px auto 10px;
  display: block;
}

.prize-item p {
  color: #900C16;
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

:deep(.history-modal),
:deep(.setting-modal) {
  .ant-modal {
    max-height: 70vh;
    
    .ant-modal-content {
      max-height: 70vh;
      display: flex;
      flex-direction: column;
      
      .ant-modal-body {
        flex: 1;
        overflow: auto;
      }
    }
  }
}

</style>
