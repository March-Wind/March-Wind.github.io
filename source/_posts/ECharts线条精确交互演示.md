---
title: ECharts 线条精确交互演示 (PC/移动端)
date: 2025-8-21 19:21:00
tags:
  - ECharts
  - JavaScript
  - 数据可视化
  - 交互设计
categories:
  - 技术分享
  - 前端开发
description: 演示透明辅助线如何实现精确的线条交互效果，支持PC端Hover和移动端点击交互
index_img: /img/echarts-logo.png
---

本文演示了如何在 ECharts 中实现精确的线条交互效果，解决了十字线经过数据点时意外触发线条高亮的问题，同时支持 PC 端和移动端的不同交互方式。

<!-- more -->

## 🎯 演示效果

<div class="demo-container">
  <div class="demo-info">
    <h3>📋 测试说明：</h3>
    <ul>
      <li><strong>蓝色线条</strong>：原始线条，宽度2px，正常显示</li>
      <li><strong>红色线条</strong>：原始线条，宽度2px，正常显示</li>
      <li><strong>透明辅助区域</strong>：宽度12px，完全透明，用于扩大交互检测</li>
      <li><strong>PC端测试重点</strong>：只有鼠标接近线条时才高亮，移动十字线经过数据点不会高亮</li>
      <li><strong>移动端测试重点</strong>：点击线条附近可高亮切换，支持取消和切换(F12切换手机模式，然后刷新)</li>
    </ul>
  </div>

  <div class="demo-highlight">
    💡 <strong>PC端测试</strong>：①鼠标hover线条附近→线条高亮 ②十字线经过→不高亮！<br />
    📱 <strong>移动端测试</strong>：①点击线条附近→高亮切换 ②再次点击→取消高亮！
  </div>

  <div id="chart" class="chart-container"></div>
</div>

## 🔧 实现原理

### 原理 1：禁用 axisPointer 触发系列强调效果

全局禁用`axisPointer`触发系列强调效果，确保十字线经过数据点时不会触发线条高亮：

```javascript
// 全局禁用axisPointer触发系列强调效果
axisPointer: {
  triggerEmphasis: false, // 关键：全局禁用axisPointer触发系列强调效果
}

// 每个线条里的triggerEmphasis: false也需要设置
series: [
  {
    // ...
    triggerEmphasis: false,
  }
]
```

### 原理 2：透明辅助线扩大交互区域

为每条线创建同名的透明辅助粗线（宽 12px），比原始线条宽 6 倍，用户看不见但能响应交互事件：

- **PC 端**：hover 辅助线 → 高亮原始线，mouseout→ 恢复正常
- **移动端**：点击辅助线/原始线 → 切换高亮状态，支持取消和切换
- **小技巧**：辅助线置于底层（z: -1），用户看不见但能响应交互
- **小技巧**：通过 dispatchAction 高亮同名的原始线条

## 💻 完整代码实现

<style>
.demo-container {
  margin: 20px 0;
}

.demo-info {
  background: #e3f2fd;
  border-left: 4px solid #2196f3;
  padding: 15px;
  margin: 20px 0;
  border-radius: 0 4px 4px 0;
}

.demo-info h3 {
  margin: 0 0 10px 0;
  color: #1976d2;
}

.demo-info ul {
  margin: 0;
  padding-left: 20px;
}

.demo-info li {
  margin: 5px 0;
  color: #555;
}

.demo-highlight {
  background: #fff3cd;
  border: 1px solid #ffeaa7;
  padding: 10px;
  border-radius: 4px;
  margin: 10px 0;
  text-align: center;
  color: #856404;
  font-weight: bold;
}

.chart-container {
  width: 100%;
  height: 500px;
  margin: 20px 0;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
}

@media (max-width: 768px) {
  .chart-container {
    height: 400px;
  }
}
</style>

<script src="https://cdn.jsdelivr.net/npm/echarts@6.0.0/dist/echarts.min.js"></script>
<script>
// 等待页面加载完成
document.addEventListener('DOMContentLoaded', function() {
  // 检查ECharts是否已加载
  if (typeof echarts === 'undefined') {
    console.error('ECharts未正确加载');
    return;
  }

  // 初始化ECharts实例
  const chartDom = document.getElementById('chart');
  if (!chartDom) {
    console.error('找不到chart容器元素');
    return;
  }

  const myChart = echarts.init(chartDom);

  // 生成示例数据
  const generateData = (base, variance) => {
    const data = [];
    for (let i = 0; i < 50; i++) {
      data.push([
        i,
        base + Math.sin(i * 0.2) * variance + Math.random() * 20 - 10,
      ]);
    }
    return data;
  };

  const data1 = generateData(100, 30); // 蓝色线数据
  const data2 = generateData(150, 40); // 红色线数据

  const option = {
    title: {
      text: '线条精确交互测试 (PC端Hover / 移动端点击)',
      left: 'center',
      textStyle: {
        fontSize: 16,
      },
    },
    // 🎯 全局axisPointer配置 - 禁用自动强调
    axisPointer: {
      triggerEmphasis: false, // 关键：全局禁用axisPointer触发系列强调效果
    },
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'cross',
        crossStyle: {
          color: '#999',
        },
        triggerEmphasis: false, // 🎯 关键：禁用axisPointer触发系列强调效果
      },
    },
    legend: {
      data: ['趋势线A', '趋势线B'],
      top: 40,
    },
    grid: {
      left: '3%',
      right: '4%',
      bottom: '3%',
      top: '15%',
      containLabel: true,
    },
    xAxis: {
      type: 'value',
      name: '时间点',
    },
    yAxis: {
      type: 'value',
      name: '数值',
    },
    series: [
      // === 原始线条A ===
      {
        name: '趋势线A',
        type: 'line',
        data: data1,
        smooth: true,
        lineStyle: {
          width: 2,
          color: '#1890ff',
        },
        itemStyle: {
          color: '#1890ff',
        },
        emphasis: {
          scale: false,
          lineStyle: {
            width: 4,
            shadowColor: 'rgba(24, 144, 255, 0.8)',
            shadowBlur: 8,
            shadowOffsetY: 2,
          },
        },
        showSymbol: false,
        hoverAnimation: true,
        triggerLineEvent: true,
        triggerEmphasis: false, // 🎯 关键：禁用被axisPointer自动强调
      },
      // === 透明辅助线A ===
      {
        name: '趋势线A', // 🎯 关键：同名！
        type: 'line',
        data: data1, // 🎯 关键：同数据！
        smooth: true,
        lineStyle: {
          width: 12, // 🚀 6倍宽度用于hover检测
          color: 'transparent', // 👻 完全透明
          opacity: 0,
        },
        itemStyle: {
          color: 'transparent',
          opacity: 0,
        },
        showInLegend: false, // 🚫 不显示在图例中
        showSymbol: false,
        hoverAnimation: false,
        triggerLineEvent: true,
        triggerEmphasis: false, // 🎯 关键：禁用被axisPointer自动强调
        z: -1, // 🎯 底层，不遮挡其他元素
      },

      // === 原始线条B ===
      {
        name: '趋势线B',
        type: 'line',
        data: data2,
        smooth: true,
        lineStyle: {
          width: 2,
          color: '#ff4d4f',
        },
        itemStyle: {
          color: '#ff4d4f',
        },
        emphasis: {
          scale: false,
          lineStyle: {
            width: 4,
            shadowColor: 'rgba(255, 77, 79, 0.8)',
            shadowBlur: 8,
            shadowOffsetY: 2,
          },
        },
        showSymbol: false,
        hoverAnimation: true,
        triggerLineEvent: true,
        triggerEmphasis: false, // 🎯 关键：禁用被axisPointer自动强调
      },
      // === 透明辅助线B ===
      {
        name: '趋势线B', // 🎯 关键：同名！
        type: 'line',
        data: data2, // 🎯 关键：同数据！
        smooth: true,
        lineStyle: {
          width: 12, // 🚀 6倍宽度用于hover检测
          color: 'transparent', // 👻 完全透明
          opacity: 0,
        },
        itemStyle: {
          color: 'transparent',
          opacity: 0,
        },
        showInLegend: false, // 🚫 不显示在图例中
        showSymbol: false,
        hoverAnimation: false,
        triggerLineEvent: true,
        triggerEmphasis: false, // 🎯 关键：禁用被axisPointer自动强调
        z: -1, // 🎯 底层，不遮挡其他元素
      },
    ],
  };

  // 设置图表配置
  myChart.setOption(option);

  // 🎯 检测是否为移动设备
  const isMobile =
    /Android|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
      navigator.userAgent
    ) || window.innerWidth <= 768;

  let currentHighlighted = null; // 记录当前高亮的系列

  // 🎯 关键事件处理：实现精确的hover联动
  myChart.on('mouseover', function (params) {
    if (isMobile) return; // 移动端跳过hover事件

    console.log('🎯 Hover事件触发:', params.seriesName);

    if (params.seriesName) {
      // 高亮所有同名系列（原始线条会被高亮）
      myChart.dispatchAction({
        type: 'highlight',
        seriesName: params.seriesName,
      });

      // 同时高亮图例
      myChart.dispatchAction({
        type: 'legendHighlight',
        name: params.seriesName,
      });

      // 在页面上显示提示
      showHoverInfo(params.seriesName, isMobile ? '点击高亮' : '高亮');
    }
  });

  myChart.on('mouseout', function (params) {
    if (isMobile) return; // 移动端跳过hover事件

    console.log('🔄 Mouse离开:', params.seriesName);

    if (params.seriesName) {
      // 取消高亮
      myChart.dispatchAction({
        type: 'downplay',
        seriesName: params.seriesName,
      });

      myChart.dispatchAction({
        type: 'legendDownplay',
        name: params.seriesName,
      });

      // 清除提示
      showHoverInfo(params.seriesName, '恢复');
    }
  });

  // 🎯 点击事件处理（PC端和移动端）
  myChart.on('click', function (params) {
    console.log('📱 点击事件触发:', params.seriesName, '移动端:', isMobile);

    if (params.seriesName) {
      // 点击到了线条
      if (isMobile) {
        // 移动端：点击切换高亮状态
        if (currentHighlighted === params.seriesName) {
          // 当前系列已高亮，取消高亮
          myChart.dispatchAction({
            type: 'downplay',
            seriesName: params.seriesName,
          });
          myChart.dispatchAction({
            type: 'legendDownplay',
            name: params.seriesName,
          });
          currentHighlighted = null;
          showMobileInfo(params.seriesName, '取消高亮');
        } else {
          // 先清除之前的高亮
          if (currentHighlighted) {
            myChart.dispatchAction({
              type: 'downplay',
              seriesName: currentHighlighted,
            });
            myChart.dispatchAction({
              type: 'legendDownplay',
              name: currentHighlighted,
            });
          }

          // 高亮新的系列
          myChart.dispatchAction({
            type: 'highlight',
            seriesName: params.seriesName,
          });
          myChart.dispatchAction({
            type: 'legendHighlight',
            name: params.seriesName,
          });
          currentHighlighted = params.seriesName;
          showMobileInfo(params.seriesName, '点击高亮');
        }
      } else {
        // PC端：点击选中图例
        myChart.dispatchAction({
          type: 'legendToggleSelect',
          name: params.seriesName,
        });
        showHoverInfo(params.seriesName, '图例切换');
      }
    } else if (isMobile && currentHighlighted) {
      // 移动端：点击空白区域，取消当前高亮
      myChart.dispatchAction({
        type: 'downplay',
        seriesName: currentHighlighted,
      });
      myChart.dispatchAction({
        type: 'legendDownplay',
        name: currentHighlighted,
      });
      showMobileInfo(currentHighlighted, '取消高亮');
      currentHighlighted = null;
    }
  });

  // 显示hover状态信息（PC端）
  function showHoverInfo(seriesName, action) {
    const info = document.querySelector('.demo-highlight');
    if (!info) return;
    
    if (action === '高亮' || action === '点击高亮') {
      info.innerHTML = `✨ <strong>${seriesName}</strong> 正在高亮显示！`;
      info.style.background = '#d4edda';
      info.style.borderColor = '#c3e6cb';
      info.style.color = '#155724';
    } else if (action === '图例切换') {
      info.innerHTML = `🔄 <strong>${seriesName}</strong> 图例状态已切换！`;
      info.style.background = '#e3f2fd';
      info.style.borderColor = '#bbdefb';
      info.style.color = '#1565c0';
    } else {
      info.innerHTML = isMobile
        ? '📱 <strong>移动端</strong>：点击线条附近→高亮切换！再次点击→取消高亮！点击空白→取消'
        : '💡 <strong>PC端</strong>：鼠标hover线条附近→高亮 / 十字线经过→不高亮！';
      info.style.background = '#fff3cd';
      info.style.borderColor = '#ffeaa7';
      info.style.color = '#856404';
    }
  }

  // 显示移动端状态信息
  function showMobileInfo(seriesName, action) {
    const info = document.querySelector('.demo-highlight');
    if (!info) return;
    
    if (action === '点击高亮') {
      info.innerHTML = `📱 <strong>${seriesName}</strong> 已高亮！再次点击可取消`;
      info.style.background = '#d4edda';
      info.style.borderColor = '#c3e6cb';
      info.style.color = '#155724';
    } else if (action === '取消高亮') {
      info.innerHTML = `📱 <strong>${seriesName}</strong> 高亮已取消！点击其他线条可高亮`;
      info.style.background = '#f8d7da';
      info.style.borderColor = '#f5c6cb';
      info.style.color = '#721c24';
    }
  }

  // 响应式处理
  window.addEventListener('resize', function () {
    if (myChart && !myChart.isDisposed()) {
      myChart.resize();
    }
  });

  // 添加说明文字
  console.log('📊 Demo加载完成！');
  console.log('🎯 测试要点：');
  if (isMobile) {
    console.log('📱 移动端模式：');
    console.log('1. 点击线条附近 → 高亮切换');
    console.log('2. 再次点击同一线条 → 取消高亮');
    console.log('3. 点击其他线条 → 切换到新线条高亮');
    console.log('4. 点击空白区域 → 取消当前高亮');
  } else {
    console.log('💻 PC端模式：');
    console.log('1. 鼠标hover线条附近 → 线条高亮');
    console.log('2. 十字线经过数据点 → 线条不高亮');
    console.log('3. 鼠标离开 → 恢复正常');
  }
  console.log('5. 透明辅助线扩大了交互检测范围（12px）');

  // 🎯 初始化提示信息
  showHoverInfo('', '初始化');
});
</script>

## 📝 技术要点总结

1. **全局配置**：通过 `axisPointer.triggerEmphasis: false` 禁用十字线触发高亮
2. **透明辅助线**：创建同名透明线条，宽度 6 倍，用于扩大交互区域
3. **设备适配**：自动检测移动设备，采用不同的交互方式
4. **状态管理**：跟踪当前高亮状态，支持切换和取消
5. **响应式设计**：图表自动适应容器大小变化

这种方案完美解决了 ECharts 线条交互的精确性问题，无论在 PC 端还是移动端都能提供优秀的用户体验！
