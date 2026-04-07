<template>
  <!-- 全屏容器：HMI 工业深色风 -->
  <div class="w-screen h-screen bg-[#020617] text-cyan-400 font-mono overflow-hidden flex flex-col select-none">
    
    <!-- 1. 顶部 Header (需求6 & 7) -->
    <header class="h-12 border-b border-cyan-900 bg-slate-900/80 flex justify-between items-center px-4 shrink-0 z-50">
      <div class="flex items-center gap-4">
        <span class="text-xl font-black italic tracking-tighter text-white">JH-HORIZON PRO</span>
        <div class="flex gap-2 text-[10px]">
          <span v-if="isTesting" class="px-2 bg-red-900/40 text-red-500 border border-red-900 animate-pulse uppercase">● Ring-Buffer: High-Speed Mode</span>
          <span v-else class="px-2 bg-cyan-900/40 text-cyan-500 border border-cyan-900 uppercase">○ Sampling: Low-Freq (10fps)</span>
        </div>
      </div>
      <div class="flex gap-6 items-center text-[10px]">
        <span>采样率 (Sampling): {{ samplingRate }}%</span>
        <span>坐标系: 最佳拟合 (Best-Fit)</span>
        <span class="opacity-50 uppercase tracking-widest">{{ currentTime }}</span>
      </div>
    </header>

    <!-- 中间主体布局 -->
    <div class="flex-1 flex overflow-hidden">
      
      <!-- 2. 左侧：元素及分析树 (需求 8) -->
      <aside class="w-60 border-r border-cyan-900 bg-slate-950/80 p-3 flex flex-col gap-4 z-40">
        <div class="flex-1">
          <h3 class="text-[10px] font-bold opacity-40 mb-3 border-b border-cyan-900 pb-1 uppercase">分析项目 (Analysis)</h3>
          <div class="space-y-1">
            <div v-for="item in ['Mises 应变分析','Tresca 应变计算','高斯曲率','主应变 K1/K2','面内旋转角']" :key="item" 
                 class="p-2 text-[11px] bg-cyan-950/20 border border-cyan-900/50 hover:border-cyan-400 cursor-pointer transition-all">
              {{ item }}
            </div>
          </div>
        </div>
        <div class="flex-1">
          <h3 class="text-[10px] font-bold opacity-40 mb-3 border-b border-cyan-900 pb-1 uppercase">几何元素创建</h3>
          <div class="grid grid-cols-2 gap-1 text-[9px]">
            <button v-for="e in ['三维点','圆柱拟合','矩形孔','拟合球','线面夹角','点面距离']" :key="e" class="p-2 border border-slate-800 hover:bg-cyan-900/50 active:scale-95 transition-all">+ {{e}}</button>
          </div>
        </div>
      </aside>

      <!-- 3. 中间：三维全场可视化视口 (需求 6 核心) -->
      <main class="flex-1 relative bg-black flex flex-col overflow-hidden">
        <!-- 3D 顶部操作按钮 -->
        <div class="absolute top-4 left-4 z-30 flex gap-3">
          <button @click="startStressTest" 
                  class="px-6 py-2 text-xs font-bold transition-all flex items-center gap-2 shadow-lg"
                  :class="isTesting ? 'bg-red-600 text-white animate-pulse' : 'bg-cyan-600 text-black'">
            {{ isTesting ? '停止应力加载' : '开始 3D 位移分析 (START TEST)' }}
          </button>
          <button @click="toggleWireframe" class="bg-slate-800 border border-cyan-600 text-cyan-400 px-4 py-2 text-[10px] uppercase">切换网格视图</button>
          <button @click="resetModel" class="bg-slate-800 border border-slate-600 text-slate-400 px-4 py-2 text-[10px] uppercase">重置数据</button>
        </div>

        <!-- 3D 侧边色谱图例 (Color Bar) -->
        <div class="absolute right-6 top-16 bottom-16 w-12 flex flex-col items-center z-20">
          <div class="text-[10px] text-red-500 font-bold mb-1">{{ (maxZ * 5).toFixed(2) }}</div>
          <div class="flex-1 w-4 bg-gradient-to-t from-blue-600 via-green-500 via-yellow-400 to-red-600 border border-white/20"></div>
          <div class="text-[10px] text-blue-500 font-bold mt-1">0.00</div>
          <div class="mt-4 -rotate-90 whitespace-nowrap text-[9px] opacity-40 uppercase tracking-tighter">位移分析 (mm)</div>
        </div>

        <!-- Three.js 容器 -->
        <div ref="threeContainer" class="flex-1 w-full h-full cursor-grab active:cursor-grabbing"></div>

        <!-- FEA 对比标签 (需求 6) -->
        <div class="absolute bottom-6 left-6 z-20 p-3 bg-black/60 border border-cyan-900 backdrop-blur-md">
          <p class="text-[10px] opacity-60 uppercase mb-1">比对模组: ABAQUS / Ansys Standard</p>
          <p class="text-xs text-yellow-500 font-bold uppercase tracking-widest italic">偏差均值: {{ (maxZ * 0.1).toFixed(4) }} mm</p>
        </div>

        <!-- 底部力学参数流 (需求 6) -->
        <div class="h-24 bg-slate-950/90 border-t border-cyan-900 grid grid-cols-4 p-2 gap-2 shrink-0 z-30">
          <div v-for="(val, label) in physicsParams" :key="label" class="flex flex-col justify-center px-4 bg-cyan-950/10 border border-cyan-900/20">
            <span class="text-[9px] opacity-40 uppercase tracking-tighter">{{label}}</span>
            <span class="text-lg font-bold text-white tabular-nums">{{val}}</span>
          </div>
        </div>
      </main>

      <!-- 4. 右侧：网格工具与坐标系 (需求 9) -->
      <aside class="w-64 border-l border-cyan-900 bg-slate-950/80 p-3 flex flex-col gap-4 z-40 text-[11px]">
        <section>
          <h3 class="text-[10px] font-bold opacity-40 mb-3 border-b border-cyan-900 pb-1 uppercase">网格修复 (Mesh Tools)</h3>
          <div class="flex flex-col gap-1">
            <button @click="simpleAlert('执行曲率自动补洞...')" class="text-left p-2 bg-slate-900 border border-slate-800 hover:border-cyan-400 transition-all">曲率补洞 (Curvature Fill)</button>
            <button class="text-left p-2 bg-slate-900 border border-slate-800 hover:border-cyan-400 transition-all">手动填补 / 边缘删除</button>
            <button class="text-left p-2 bg-slate-900 border border-slate-800 hover:border-cyan-400 transition-all">修复钉状物 (Spikes)</button>
            <button class="text-left p-2 bg-slate-900 border border-slate-800 hover:border-cyan-400 transition-all">修复折射边</button>
            <button @click="simpleAlert('封装完成，STL已导出')" class="mt-2 text-center p-2 bg-cyan-900/40 text-cyan-400 border border-cyan-600 font-bold uppercase">封闭网格 STL 输出</button>
          </div>
        </section>
        
        <section class="flex-1">
          <h3 class="text-[10px] font-bold opacity-40 mb-3 border-b border-cyan-900 pb-1 uppercase">坐标转换与找正</h3>
          <div class="space-y-1">
            <div v-for="t in ['3-2-1 找正模式','参考点对齐 (RPS)','手动平移旋转','最佳拟合 (Best-Fit)']" :key="t" 
                 class="p-2 border border-slate-800 hover:bg-cyan-950/50 cursor-pointer">
              ● {{ t }}
            </div>
          </div>
        </section>
      </aside>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

// --- 基础状态 ---
const isTesting = ref(false);
const currentTime = ref('');
const samplingRate = ref(63);
const maxZ = ref(0);
const threeContainer = ref(null);
const physicsParams = reactive({
  "Mises Strain": "0.0000",
  "Tresca Strain": "0.0000",
  "X-Displacement": "0.0000",
  "Gaussian Curvature": "0.0000",
  "Angular Accel": "0.00",
  "Strain Rate": "0.000",
  "Thickness Reduction": "0.00",
  "Shear Angle": "0.00"
});

// --- Three.js 变量 ---
let scene, camera, renderer, geometry, mesh, controls, animationId;

// 初始化 3D 环境
const initThree = () => {
  scene = new THREE.Scene();
  const width = threeContainer.value.clientWidth;
  const height = threeContainer.value.clientHeight;
  
  camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 1000);
  camera.position.set(3, 4, 6);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(window.devicePixelRatio);
  threeContainer.value.appendChild(renderer.domElement);

  // 创建一个高密度的网格平面 (模拟我们要分析的板材)
  geometry = new THREE.PlaneGeometry(5, 5, 128, 128);
  
  // 材质使用顶点着色 (Vertex Colors) 来实现色谱热力图
  const material = new THREE.MeshStandardMaterial({
    side: THREE.DoubleSide,
    vertexColors: true,
    wireframe: false,
    roughness: 0.3,
    metalness: 0.5
  });

  mesh = new THREE.Mesh(geometry, material);
  mesh.rotation.x = -Math.PI / 2.5; // 斜放一点，更有立体感
  scene.add(mesh);

  // 灯光系统
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
  scene.add(ambientLight);
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 10, 5);
  scene.add(dirLight);

  // 辅助网格 (HMI 感觉)
  const grid = new THREE.GridHelper(10, 20, 0x0891b2, 0x022c37);
  grid.position.y = -1;
  scene.add(grid);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  animate();
};

// 核心：动态应变位移模拟引擎
const updatePhysicsDeformation = () => {
  if (!isTesting.value) return;

  const positions = geometry.attributes.position;
  const colors = [];
  const time = Date.now() * 0.0015;
  let localMaxZ = 0;

  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);
    
    // 物理算法模拟：根据离中心点的距离产生位移(Z轴隆起)
    const dist = Math.sqrt(x * x + y * y);
    // 产生一个随时间波动的隆起效果，模拟受力变形
    const z = Math.abs(Math.sin(dist * 1.5 - time) * (2.5 - dist) * 0.4); 
    
    positions.setZ(i, z);
    if (z > localMaxZ) localMaxZ = z;

    // 色谱映射 (Color Mapping): Z越高越红，越低越蓝
    const color = new THREE.Color();
    // 0.7 是蓝色(低), 0 是红色(高)
    const hue = Math.max(0, 0.7 - z * 0.8); 
    color.setHSL(hue, 1, 0.5);
    colors.push(color.r, color.g, color.b);
  }

  geometry.attributes.position.needsUpdate = true;
  geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));
  
  // 同步更新 UI 参数
  maxZ.value = localMaxZ;
  physicsParams["Mises Strain"] = (localMaxZ * 0.48).toFixed(4);
  physicsParams["Tresca Strain"] = (localMaxZ * 0.51).toFixed(4);
  physicsParams["X-Displacement"] = (localMaxZ * 1.25).toFixed(4);
  physicsParams["Gaussian Curvature"] = (Math.sin(time) * 0.002).toFixed(4);
  physicsParams["Angular Accel"] = (localMaxZ * 25).toFixed(2);
  physicsParams["Thickness Reduction"] = (localMaxZ * 0.15).toFixed(2) + "%";
};

const animate = () => {
  animationId = requestAnimationFrame(animate);
  updatePhysicsDeformation();
  controls.update();
  renderer.render(scene, camera);
};

// 交互函数
const startStressTest = () => { isTesting.value = !isTesting.value; };
const toggleWireframe = () => { mesh.material.wireframe = !mesh.material.wireframe; };
const resetModel = () => {
  isTesting.value = false;
  const positions = geometry.attributes.position;
  for (let i = 0; i < positions.count; i++) { positions.setZ(i, 0); }
  geometry.attributes.position.needsUpdate = true;
  maxZ.value = 0;
  Object.keys(physicsParams).forEach(k => physicsParams[k] = "0.0000");
};
const simpleAlert = (m) => alert(m);

// 生命周期
onMounted(() => {
  initThree();
  setInterval(() => { currentTime.value = new Date().toLocaleString(); }, 1000);
  window.addEventListener('resize', () => {
    camera.aspect = threeContainer.value.clientWidth / threeContainer.value.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(threeContainer.value.clientWidth, threeContainer.value.clientHeight);
  });
});

onUnmounted(() => {
  cancelAnimationFrame(animationId);
  renderer.dispose();
});
</script>

<style>
/* 针对工业平板触摸优化的微动效 */
button:active { transform: translateY(1px); filter: brightness(1.2); }
canvas { outline: none; }
</style>