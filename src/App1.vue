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
       backgroundColor: backgroundColor }" @click="incrementCounter">
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
    <div ref="tableChart" style="width: 600px; height: 400px;"></div>
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
      backgroundHeight: window.innerHeight * 0.4, // 背景高度

      generationsData: [], // 用于存储每一代的形状颜色统计数据
      colorRange: [] // 存储不同颜色数量范围的数据
    };
  },
  mounted() {
    this.initChart();
    this.initTableChart();
    this.generateNonOverlappingShapes(10); // 初始化页面时生成不重叠的形状
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
      this.counter++;
    },
    deleteShape(index) {
      const deletedColor = this.shapes[index].color;
      this.shapes.splice(index, 1);

      // 更新颜色数量表
      if (!this.colorCount[deletedColor]) {
        this.$set(this.colorCount, deletedColor, 0);
      }
      this.colorCount[deletedColor]--;
        // 当数量达到阈值时，生成后代
      if (this.colorCount[deletedColor] === this.threshold) {
        this.generateOffspring();
      }
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

      this.generationsData = this.calculateGenerationsData(); // 计算新的 generationsData 数据
      this.updateTableChart(); // 更新图表数据

    },
    deleteShape(index) {
      const deletedColor = this.shapes[index].color;
      this.shapes.splice(index, 1);

      if (!this.colorCount[deletedColor]) {
        this.$set(this.colorCount, deletedColor, 0);
      }
      this.colorCount[deletedColor]--;

      if (this.colorCount[deletedColor] === this.threshold) {
        this.generateOffspring();
      }
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
    },

  watch: {
    generations: {
      handler() {
        this.updateChart(); // 监听 generations 数据变化，更新图表
      },
      deep: true
    }
  },

  initTableChart(){
      this.tableChart = echarts.init(this.$refs.tableChart);
      this.tableChart.setOption({
        title: {
          text: '不同颜色数量范围随着代数的变化'
        },
        tooltip: {
          position: 'top'
        },
        animation: false,
        grid: {
          height: '50%',
          y: '10%'
        },
        xAxis: {
          type: 'category',
          data: [] // x 轴数据，用于展示颜色数量范围
        },
        yAxis: {
          type: 'category',
          data: [] // y 轴数据，用于展示代数
        },
        visualMap: {
          min: 0,
          max: 20, // 最大代数范围，可根据实际情况调整
          calculable: true,
          orient: 'horizontal',
          left: 'center',
          bottom: '15%',
          inRange: {
            color: ['#FFFFFF', '#FF0000'] // 颜色映射范围
          }
        },
        series: [{
          name: '颜色数量范围',
          type: 'heatmap',
          data: [] // 表格数据
        }]
      });
    }
  },
  // 更新 ECharts 图表数据
  updateTableChart() {
    const data = [];
      this.generationsData.forEach((genData, genIndex) => {
        genData.forEach((colorCount, colorIndex) => {
          data.push([colorIndex, genIndex, colorCount]);
        });
      });

      this.tableChart.setOption({
        xAxis: {
          data: this.colorRange.map(item => `${item[0]} - ${item[1]}`) // 更新 x 轴数据
        },
        yAxis: {
          data: this.generationsData.map((_, index) => `第${index + 1}代`) // 更新 y 轴数据
        },
        series: [{
          data // 更新表格数据
        }]
      });
    
  },

  calculateGenerationsData() {
    const minColors = this.colors.length; // 最小颜色数量
    const maxColors = 2 * this.colors.length; // 最大颜色数量

    const generationsData = [];
    for (let i = 0; i < this.generations.length; i++) {
      const shapes = this.generations[i]; // 获取当前代的形状数据
      const colorCount = new Array(this.colors.length).fill(0); // 初始化颜色数量统计数组

      shapes.forEach(shape => {
        const colorIndex = this.colors.indexOf(shape.color);
        if (colorIndex !== -1) {
          colorCount[colorIndex]++;
        }
      });

      const colorCountInRange = colorCount.reduce((acc, count) => {
        if (count >= minColors && count <= maxColors) {
          return acc + 1;
        }
        return acc;
      }, 0);

      generationsData.push(colorCountInRange); // 记录当前代中颜色数量在范围内的数量
    }

    return generationsData;
  },

  watch: {
    generationsData: {
      handler() {
        this.updateTableChart(); // 监听 generationsData 数据变化，更新表格
      },
      deep: true
    }
  }

};

</script>

<style>
.container {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
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

