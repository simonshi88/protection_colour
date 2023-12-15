<template>
  <div>
    <p>
      Quantity Threshold: 
      <input type="number" v-model="threshold" />
      <button @click="generateOffspring">Generate Offspring</button>
    </p>

    <p>
      <button @click="clearAll">Clear All</button>
    </p>

    <div class="container">
      <div class="background" :style="{ width: backgroundWidth + 'px', height: backgroundHeight + 'px',
       backgroundColor: backgroundColor }" >
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


    <p>Click Count: {{ counter }}</p>
    <table>
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
    </table>

    <!-- 添加一个 div 用于渲染 ECharts 图表 -->
    <div ref="chart" style="width: 600px; height: 400px;"></div>

  </div>
</template>

<script>
import * as echarts from 'echarts';

export default {
  data() {
    return {
      colors: ['red', 'blue', 'green', 'yellow', 'purple','black'],
      currentColor: '',
      backgroundColor: 'grey', // 添加一个用于存储背景颜色的变量
      shapes: [],
      counter: 0,
      colorCount: {},
      threshold: 5,
      generations: [], // 用于记录每一代的 shapes

      shapeSize: 20, // 形状大小
      backgroundWidth: window.innerWidth * 0.9, // 背景宽度
      backgroundHeight: window.innerHeight * 0.6, // 背景高度

      generationsData: [], // 用于存储每一代的形状颜色统计数据
      colorRange: [] // 存储不同颜色数量范围的数据
    };
  },
  mounted() {
    this.initChart();
    this.generateNonOverlappingShapes(10); // 初始化页面时生成不重叠的形状
  },

  created() {
    console.log(this.generationsData);
  },

  methods: {

    generateShapes(quantity) {
      const screenWidth = backgroundWidth;
      const screenHeight = backgroundHeight;

      for (let i = 0; i < quantity; i++) {
        const shape = {
          id: i,
          color: this.colors[Math.floor(Math.random() * this.colors.length)],
          top: Math.random() * screenHeight,
          left: Math.random() * screenWidth
        };

        if (shape.top > screenHeight - this.shapeSize) {
          shape.top -= this.shapeSize;
        }
        if (shape.left > screenWidth - this.shapeSize) {
          shape.left -= this.shapeSize;
        }

        this.shapes.push(shape);
      }
    },
    
    incrementCounter() {
      if(this.counter >= this.threshold - 1){
        this.counter = 0;
        this.generateOffspring();
      }else{
        this.counter++;
      }
    },

    deleteShape(index) {
      const deletedColor = this.shapes[index].color;
      this.shapes.splice(index, 1);

      this.incrementCounter();
      if (!this.colorCount[deletedColor]) {
        this.colorCount[deletedColor] = 0; // 直接赋值
      }
        this.colorCount[deletedColor]--;
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

      for (let i = 0; i < quantity; i++) {
        const { top, left } = this.getNonOverlappingPosition(existingPositions);
        existingPositions.push({ top, left, bottom: top + this.shapeSize, right: left + this.shapeSize });
        offspring.push({
          id: i + 1,
          color: this.colors[Math.floor(Math.random() * this.colors.length)],
          top,
          left
        });
      }

      this.shapes = offspring;
      this.generations.push(JSON.parse(JSON.stringify(this.shapes)));
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
    },


    clearAll() {
      this.shapes = [];
      this.counter = 0;
      this.colorCount = {};
      this.generations = [];
      this.generateNonOverlappingShapes(10); // 初始生成不重叠的形状
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
          data: this.colors // x 轴使用颜色作为类目
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
      const lastGeneration = this.generations[this.generations.length - 1]; // 获取最后一代的数据

      if (lastGeneration) {
        this.colors.forEach(color => {
          const count = lastGeneration.filter(shape => shape.color === color).length; // 统计每种颜色的数量
          data.push(count);
        });
      }

      this.chart.setOption({
        series: [{
          data // 更新数据
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
  height: 50%;
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
  border: 1px solid transparent;
  border-radius: 3px;
}
</style>

