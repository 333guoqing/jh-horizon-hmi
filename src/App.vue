<template>
  <div class="h-screen w-screen bg-[#0B141A] text-cyan-400 font-mono overflow-hidden select-none">
    
    <!-- 1. 登录页面 -->
    <transition name="fade">
      <div v-if="!isLoggedIn" class="absolute inset-0 z-50 flex items-center justify-center bg-[#0B141A]">
        <div class="p-8 border-2 border-cyan-500 bg-slate-900 shadow-[0_0_30px_rgba(6,182,212,0.3)] w-[400px]">
          <h1 class="text-3xl font-bold mb-8 text-center tracking-widest">中科捷航 · 视界</h1>
          <input v-model="loginForm.user" type="text" placeholder="账号" class="w-full bg-black border border-cyan-800 p-3 mb-4 outline-none focus:border-cyan-400 text-white" />
          <input v-model="loginForm.pass" type="password" placeholder="密码" class="w-full bg-black border border-cyan-800 p-3 mb-6 outline-none focus:border-cyan-400 text-white" />
          <button @click="handleLogin" class="w-full bg-cyan-600 hover:bg-cyan-500 text-black font-bold py-3 transition-colors uppercase">进入系统</button>
        </div>
      </div>
    </transition>

    <!-- 2. 主监控面板 -->
    <div v-if="isLoggedIn" class="h-full flex flex-col p-4">
      <!-- 顶部状态栏 -->
      <header class="flex justify-between items-center border-b border-cyan-900 pb-2 mb-4 bg-slate-900/50 px-4 py-2">
        <div class="flex items-baseline gap-4">
          <span class="text-2xl font-black italic tracking-tighter">JH-HORIZON</span>
          <span class="text-xs opacity-50 uppercase font-normal">Industrial Monitor v4.0</span>
        </div>
        <div class="flex gap-6 items-center">
          <div class="flex items-center gap-2">
            <span class="w-2 h-2 rounded-full bg-green-500 animate-pulse"></span>
            <span class="text-xs">PLC CONNECTED</span>
          </div>
          <span class="text-white text-sm opacity-80">{{ currentTime }}</span>
          <span class="text-xs border border-cyan-800 px-2 py-1 text-cyan-700">ADMIN</span>
        </div>
      </header>

      <!-- 设备网格 (响应式三栏) -->
      <main class="flex-1 grid grid-cols-1 md:grid-cols-3 gap-6 overflow-hidden">
        <div v-for="(device, dIdx) in devices" :key="dIdx" 
             class="border border-cyan-900 bg-slate-900/30 flex flex-col p-4 relative group">
          <!-- 装饰角 -->
          <div class="absolute top-0 left-0 w-3 h-3 border-t-2 border-l-2 border-cyan-500"></div>
          <div class="absolute bottom-0 right-0 w-3 h-3 border-b-2 border-r-2 border-cyan-500"></div>

          <!-- 设备标题 -->
          <div class="flex justify-between items-center mb-6">
            <h2 class="text-lg font-bold border-l-4 border-cyan-500 pl-3 uppercase">{{ device.name }}</h2>
            <div class="text-[10px] bg-cyan-950 px-2 py-1 text-cyan-400 border border-cyan-800">MOD: AUTO</div>
          </div>

          <!-- 数值网格 -->
          <div class="grid grid-cols-2 gap-3 mb-6">
            <div v-for="(param, pIdx) in device.params" :key="pIdx" 
                 @click="openKeypad(dIdx, pIdx)"
                 class="bg-black/60 p-3 rounded cursor-pointer hover:bg-cyan-900/20 transition-colors border border-transparent hover:border-cyan-800">
              <div class="text-[10px] opacity-50 uppercase mb-1">{{ param.label }} ({{ param.unit }})</div>
              <div class="flex justify-between items-baseline">
                <span class="text-3xl text-white font-bold leading-none">{{ param.pv.toFixed(1) }}</span>
                <div class="text-[10px] text-cyan-600 flex flex-col items-end">
                  <span>SV</span>
                  <span class="font-bold">{{ param.sv }}</span>
                </div>
              </div>
            </div>
          </div>

          <!-- 实时图表容器 -->
          <div class="flex-1 min-h-[180px] mt-auto" :id="'chart-' + dIdx"></div>
        </div>
      </main>

      <!-- 底部导航 (对应你图纸上的按钮) -->
      <footer class="mt-6 flex gap-3 h-12">
        <button v-for="tab in ['老化箱', '干燥箱', '恒温恒湿', '硬度记录']" :key="tab"
                class="px-8 border border-cyan-900 text-sm hover:bg-cyan-900/40 transition-colors uppercase tracking-widest"
                :class="{'bg-cyan-900/40 border-cyan-400': tab === '老化箱'}">
          {{ tab }}
        </button>
      </footer>
    </div>

    <!-- 3. 触摸屏专用数字键盘 (弹窗) -->
    <div v-if="showKeypad" class="fixed inset-0 z-[100] flex items-center justify-center bg-black/80 backdrop-blur-sm">
      <div class="bg-slate-900 border-2 border-cyan-500 p-6 w-80">
        <div class="text-xs opacity-50 mb-1 uppercase">设定 {{ editingParam.label }}</div>
        <div class="bg-black text-3xl p-4 text-center font-bold text-white mb-4 border border-cyan-900">{{ tempInput }}</div>
        <div class="grid grid-cols-3 gap-2">
          <button v-for="n in [1,2,3,4,5,6,7,8,9,'.',0]" :key="n" @click="appendNum(n)" class="bg-slate-800 p-4 text-xl hover:bg-cyan-700 transition-colors">{{ n }}</button>
          <button @click="tempInput = ''" class="bg-red-900/50 p-4 text-xl hover:bg-red-700">C</button>
          <button @click="closeKeypad" class="col-span-1 bg-slate-700 p-4">取消</button>
          <button @click="confirmSV" class="col-span-2 bg-cyan-600 text-black font-bold p-4 text-xl">确认设定</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, nextTick } from 'vue';
import * as echarts from 'echarts';

// --- 状态管理 ---
const isLoggedIn = ref(false);
const loginForm = reactive({ user: 'admin', pass: '123456' });
const currentTime = ref('');
const showKeypad = ref(false);
const tempInput = ref('');
const editingTarget = reactive({ d: 0, p: 0 });
const editingParam = ref({});

// --- 设备数据定义 (完全对应你的纸上手稿) ---
const devices = reactive([
  {
    name: '老化箱 Aging',
    params: [
      { label: '温度', pv: 25.0, sv: 85.0, unit: '℃' },
      { label: '湿度', pv: 40.0, sv: 60.0, unit: '%RH' },
      { label: '波长', pv: 340.0, sv: 340.0, unit: 'nm' }
    ]
  },
  {
    name: '干燥箱 Drying',
    params: [
      { label: '温度', pv: 25.0, sv: 120.0, unit: '℃' },
      { label: '时间', pv: 0, sv: 3600, unit: 'SEC' }
    ]
  },
  {
    name: '恒温恒湿 Chamber',
    params: [
      { label: '温度', pv: 25.0, sv: 45.0, unit: '℃' },
      { label: '湿度', pv: 45.0, sv: 85.0, unit: '%RH' }
    ]
  }
]);

const charts = []; // 存储 ECharts 实例

// --- 核心模拟引擎 (假数据跑起来) ---
const runSimulation = () => {
  devices.forEach((dev, dIdx) => {
    dev.params.forEach((param, pIdx) => {
      // PV 追随 SV 的简单算法
      const diff = param.sv - param.pv;
      if (Math.abs(diff) > 0.1) {
        param.pv += diff * 0.05 + (Math.random() - 0.5) * 0.3; // 逼近过程
      } else {
        param.pv += (Math.random() - 0.5) * 0.1; // 稳定后的微抖动
      }
      
      // 更新图表
      if (charts[dIdx]) {
        const option = charts[dIdx].getOption();
        const now = new Date();
        const timeStr = `${now.getHours()}:${now.getMinutes()}:${now.getSeconds()}`;
        
        // 假设图表显示第一项参数的曲线
        if (pIdx === 0) {
          option.series[0].data.push([timeStr, param.pv.toFixed(1)]);
          if (option.series[0].data.length > 30) option.series[0].data.shift();
          charts[dIdx].setOption(option);
        }
      }
    });
  });
};

// --- 交互方法 ---
const handleLogin = () => {
  if (loginForm.user === 'admin' && loginForm.pass === '123456') {
    isLoggedIn.value = true;
    nextTick(initCharts); // 登录成功后初始化图表
  } else {
    alert('账号或密码错误');
  }
};

const openKeypad = (d, p) => {
  editingTarget.d = d;
  editingTarget.p = p;
  editingParam.value = devices[d].params[p];
  tempInput.value = '';
  showKeypad.value = true;
};

const appendNum = (n) => tempInput.value += n;
const closeKeypad = () => showKeypad.value = false;
const confirmSV = () => {
  if (tempInput.value) {
    devices[editingTarget.d].params[editingTarget.p].sv = parseFloat(tempInput.value);
  }
  closeKeypad();
};

// --- ECharts 初始化 ---
const initCharts = () => {
  devices.forEach((_, idx) => {
    const el = document.getElementById('chart-' + idx);
    if (!el) return;
    const chart = echarts.init(el);
    const option = {
      grid: { top: 10, left: 35, right: 10, bottom: 25 },
      xAxis: { type: 'category', axisLine: { lineStyle: { color: '#164e63' } }, splitLine: { show: false } },
      yAxis: { type: 'value', axisLine: { show: false }, splitLine: { lineStyle: { color: '#0c4a6e', type: 'dashed' } }, axisLabel: { fontSize: 10, color: '#0891b2' } },
      series: [{
        name: 'PV',
        type: 'line',
        showSymbol: false,
        smooth: true,
        lineStyle: { color: '#22d3ee', width: 2 },
        areaStyle: { color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [{ offset: 0, color: 'rgba(34, 211, 238, 0.3)' }, { offset: 1, color: 'transparent' }]) },
        data: []
      }]
    };
    chart.setOption(option);
    charts[idx] = chart;
  });
};

// --- 生命周期 ---
onMounted(() => {
  setInterval(() => { currentTime.value = new Date().toLocaleString(); }, 1000);
  setInterval(() => { if (isLoggedIn.value) runSimulation(); }, 1000);
  window.addEventListener('resize', () => charts.forEach(c => c.resize()));
  window.oncontextmenu = (e) => e.preventDefault();

});
</script>

<style>
/* 登录渐变 */
.fade-enter-active, .fade-leave-active { transition: opacity 0.5s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
::-webkit-scrollbar { width: 0; height: 0; }
</style>