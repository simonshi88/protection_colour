<template>
  <div>
    <p>
      捕食多少次后生成下一代: 
      <input type="number" v-model="threshold" />
      <button @click="echarts.generateOffspring">直接生成后代</button>
    </p>
    <p>
      初始数量: 
      <input v-model="initialQuantity"> 
      <button @click="clearAll">初始确认</button>
    </p>
    <p>已捕食次数（达到下一代时清零）: {{ counter }}</p>
    <div class="container">
      <div class="background" :style="{ width: backgroundWidth + 'px', height: backgroundHeight + 'px',
       backgroundColor: backgroundColor}" >
        <div
          v-for="(shape, index) in shapes"
          :key="shape.id"
          class="shape"
          :style="{
            backgroundColor: shape.color,
            borderColor: 'white',
            width: shapeSize + 'px',
            height: shapeSize + 'px',
            top: shape.top + 'px',
            left: shape.left + 'px'
          }"
          @click="deleteShape(index)"
        ></div>
      </div>
    </div>
    
    <br>
    <div style="display: flex; justify-content: left;">
      <div ref="chart" style="width: 600px; height: 400px;"></div>
      
    </div>
    <!-- 添加一个 div 用于渲染 ECharts 图表 -->
    <div style="display: flex; justify-content: left;">
      <table >
      <thead>
        <tr>
          <th>代数 </th>
          <th v-for="color in colorsName" :key="color">{{ color }} </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(generation, index) in generationsData" :key="index">
          <td>{{ index + 1 }}</td>
          <td v-for="count in generation">{{ count }}</td>
          <br>
          <!-- 
            <td v-for="(count, color) in generationAdjusted[index]" :key="color">{{ count }}</td>
          -->
        </tr>
      </tbody>
    </table>
    </div>


    <!-- <table>
      <thead>
        <tr>
          <th>颜色</th>
          <th>数量</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(count, color) in colorCount" :key="color">
          <td>{{ color }}</td>
          <td>{{ count }}</td>
        </tr>
      </tbody>
    </table> -->


  </div>

</template>

<script>
import * as echarts from 'echarts';
import { Photoshop } from 'vue-color'//有6中风格，用哪种直接引用对应的名字就行

export default {
  data() {
    return {
      colors: ['#dc6c0dff', '#0d74dcff', '#3CA73C', '#dcdc0dff'],
      colorsName: ['橙色', '蓝色', '绿色', '黄色'],
      currentColor: '',
      backgroundColor: '#279527ff',// 添加一个用于存储背景颜色的变量
      shapes: [],
      counter: 0,
      colorCount: {},
      threshold: 45,
      generations: [], // 用于记录每一代的 shapes
      initialQuantity: 60,
      shapeSize: 40, // 形状大小
      backgroundWidth: window.innerWidth * 0.85, // 背景宽度
      backgroundHeight: window.innerHeight * 0.9, // 背景高度

      generationsData: [], // 用于存储每一代的形状颜色统计数据
    };
  },
  mounted() {
    this.initChart();
    this.generateNonOverlappingShapes(this.initialQuantity); // 初始化页面时生成不重叠的形状

    this.updateChart(); // 更新图表
  },

  computed: {
  generationAdjusted() {
    return this.generationsData.map(generation => {
      const adjustedGeneration = {};
      for (const color in generation) {
        adjustedGeneration[color] = generation[color] - (this.colorCount[color] || 0);
      }
      return adjustedGeneration;
    });
  }
},

  components: {
    'photoshop-picker': Photoshop
  },

  methods: {

    incrementCounter() {
      if(this.counter >= this.threshold - 1){
        this.counter = 0;
        this.colorCount = this.getCurrentColorCounts();
        this.generateOffspring();
      }else{
        this.counter++;
      }
    },

    deleteShape(index) {
      const deletedColor = this.shapes[index].color;
      this.shapes.splice(index, 1);

      const existingPositions = this.shapes.map(shape => ({
        top: shape.top,
        left: shape.left,
        bottom: shape.top + this.shapeSize,
        right: shape.left + this.shapeSize
      }));

      this.shapes.forEach(shape => {

          const { top, left } = this.getNonOverlappingPosition(existingPositions);
          existingPositions.push({ top, left, bottom: top + this.shapeSize, right: left + this.shapeSize});
          shape.top = top;
          shape.left = left;
        
      });

      this.incrementCounter();
      // if (!this.colorCount[deletedColor]) {
      //   this.colorCount[deletedColor] = 0; // 直接赋值
      // }
      //   this.colorCount[deletedColor]--;
      },

    

      
    // 确保新位置不重叠的函数
    getNonOverlappingPosition(existingPositions) {
      let newTop, newLeft;
      do {
        newTop = Math.random() * (this.backgroundHeight - this.shapeSize);
        newLeft = Math.random() * (this.backgroundWidth - this.shapeSize);
      } while (
        existingPositions.some(
          pos =>
            !(newTop + this.shapeSize < pos.top || newTop > pos.bottom || newLeft + this.shapeSize < pos.left || newLeft > pos.right)
        )
      );
      return { top: newTop, left: newLeft };
    },

    // 生成不重叠的形状
    generateNonOverlappingShapes(quantity) {
      const offspring = [];
      const existingPositions = [];

      const colorsPerType = Math.floor(quantity / this.colors.length);

      for (let i = 0; i < quantity; i++) {
        const colorIndex = Math.floor(i / colorsPerType);
        const color = this.colors[colorIndex];

        const { top, left } = this.getNonOverlappingPosition(existingPositions);
        existingPositions.push({ top, left, bottom: top + this.shapeSize, right: left + this.shapeSize });
        offspring.push({
          id: i + 1,
          color: color,
          top,
          left
        });
      }

      this.shapes = offspring;
      this.generationsData.push(this.calculateColorCount()); // 将当前代的形状颜色统计数据存储到 generationsData 数组中
    },

    calculateColorCount() {
      const colorCount = new Array(this.colors.length).fill(0);

      this.shapes.forEach(shape => {
        const colorIndex = this.colors.indexOf(shape.color);
        if (colorIndex !== -1) {
          colorCount[colorIndex]++;
        }
      });

      return colorCount;
    },

    getCurrentColorCounts() {
      const colorCounts = {};

      this.shapes.forEach(shape => {
        const color = shape.color;
        if (!colorCounts[color]) {
          colorCounts[color] = 0;
        }
        colorCounts[color]++;
      });

      return colorCounts;
    },

    generateOffspring() {
      const offspring = [];
      const existingPositions = this.shapes.map(shape => ({
        top: shape.top,
        left: shape.left,
        bottom: shape.top + this.shapeSize,
        right: shape.left + this.shapeSize
      }));

      this.shapes.forEach(shape => {
        for (let i = 0; i < 3; i++) {
          const { top, left } = this.getNonOverlappingPosition(existingPositions);
          offspring.push({
            id: this.shapes.length + offspring.length + 1,
            color: shape.color,
            top,
            left
          });
          existingPositions.push({ top, left, bottom: top + this.shapeSize, right: left + this.shapeSize});
        }
      });

      this.shapes = [...this.shapes, ...offspring];
      this.generations.push(JSON.parse(JSON.stringify(this.shapes)));

      this.generationsData.push(this.calculateColorCount()); // 存储新一代的形状颜色统计数据

      // 在生成后代后更新图表数据
      this.updateChart();
    },


    clearAll() {
      this.shapes = [];
      this.counter = 0;
      this.colorCount = {};
      this.generations = [];
      this.generationsData = [];
      this.generateNonOverlappingShapes(this.initialQuantity); // 初始生成不重叠的形状
    },

    // 初始化 ECharts 图表
    initChart() {
      this.chart = echarts.init(this.$refs.chart);
      this.chart.setOption({
        title: {
          text: '每一代点击形状的颜色统计'
        },
        xAxis: {
          type: 'category',
          data: this.colorsName// x 轴使用颜色作为类目
        },
        yAxis: {
          type: 'value'
        },
        series: [{
          name: '点击次数',
          type: 'bar',
          data: [] // 初始数据为空
        }]
      });
     
    },
    // 更新 ECharts 图表数据
    updateChart() {
      const data = [];
      const lastGenerationData = this.generationsData[this.generationsData.length - 1];

      this.chart.setOption({
        series: [{
          data: lastGenerationData  // 更新数据
        }]
      });

    }
  },
  watch: {
    generations: {
      handler() {
        this.updateChart(); // 监听 generations 数据变化，更新图表
      },
      deep: true
    }
  }
  
};

</script>

<style>
.container {
  width: 100%;
  height: 60%;
  display: flex;
  justify-content: center;
  align-items: left;
}

.background {
  width: 60vw;
  height: 60vh;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  
}

.shape {
  position: absolute;
  width: 30px;
  height: 30px;
  border: 1px;
  border-radius: 10px;
}

body{
              margin: 0;
              padding: 0;
          }

</style>

