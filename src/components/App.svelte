<script>
  import { onMount } from 'svelte';
  import { select, geoPath, geoNaturalEarth1, json, zoom, zoomIdentity, csv, scaleSequential, interpolateRgb} from 'd3';
  import { feature } from 'topojson-client';
  import * as d3 from 'd3';

  let showResetButton = false;
  let zoomBehavior;
  let countryEmissions = {};
  let colorScale;

  // 饼状图的数据存储
  let countryEnergyData = {};
  // 提示弹出的默认状态
  let showInfo = false;
  // 数据描述的默认状态
  let showGlobeInfo = false;

  //项目运行时的代码存储区域，读取csv和json的区域
  onMount(() => {
    // 初始化 SVG 和 Projection
    const width = window.innerWidth;
    const height = window.innerHeight;
    const scale = Math.min(width, height) / 2.95; // 换成比例计算地图缩放情况
    const svg = select('svg');
    const projection = geoNaturalEarth1().scale(scale).translate([width / 2 - 18, height / 2 + 15]);
    const pathGenerator = geoPath().projection(projection);

    const tooltip = d3.select("#tooltip"); // 创建tooltip

    zoomBehavior = zoom().scaleExtent([1, 8]).translateExtent([[0, 0], [width, height]])
      .on('zoom', (event) => {
        svg.selectAll('path').attr('transform', event.transform);
      });

    svg.call(zoomBehavior);

    // 加载 CSV 数据
    csv('energy_consumption_2018.csv').then(data => {
      const maxEmission = data.reduce((max, d) => Math.max(max, +d.greenhouse_gas_emissions), 0);
      // 自定义颜色范围
      colorScale = scaleSequential([0, maxEmission], t => interpolateRgb("#FDEBD0", "#A04000")(t));

      data.forEach(d => {
        // 存储温室气体排放数据
        countryEmissions[d.country] = +d.greenhouse_gas_emissions;

        // 存储能源消耗数据
        countryEnergyData[d.country] = {
          //Renewable类：
          Biofuel: +d.biofuel_consumption,
          Solar: +d.solar_consumption,
          Hydro: +d.hydro_consumption,
          Wind: +d.wind_consumption,
          //Fossil类：
          Coal: +d.coal_consumption,
          Gas: +d.gas_consumption,
          Oil: +d.oil_consumption,
          //low carbon类
          Nuclear: +d.nuclear_consumption,
        };
      });
    });

    // 加载JSON数据
    json('countries-110m.json').then(data => {
      const countries = feature(data, data.objects.countries);
      svg.selectAll('path')
        .data(countries.features)
        .join('path')
        .attr('d', pathGenerator)
        .attr('fill', d => countryEmissions[d.properties.name] ? colorScale(countryEmissions[d.properties.name]) : '#D5DBDB') // 默认颜色
        
        //-----鼠标覆盖行为-----//
        .on('mouseover', function(event, d) {
          select(this).style('fill', '#566573');
          tooltip.style("opacity", 0.8)
                .html(d.properties.name + "<br/>Greenhouse Gas Emissions: " + (countryEmissions[d.properties.name] + " MtCO2e" || "N/A"))
                .style("left", (event.pageX + 20) + "px") // 信息距离鼠标的距离
                .style("top", (event.pageY - 51) + "px")
                .style("border-radius", "10px"); // 设置圆角
        })
        //-----鼠标移动行为-----//
        .on('mousemove', function(event) {
          tooltip.style("left", (event.pageX + 20) + "px") // 信息距离鼠标的距离
                .style("top", (event.pageY - 51) + "px");
        })
        //-----鼠标离开行为-----//
        .on('mouseout', function(event, d) {
          select(this).style('fill', countryEmissions[d.properties.name] ? colorScale(countryEmissions[d.properties.name]) : '#D5DBDB');
          tooltip.style("opacity", 0);
        })

        //-----鼠标点击行为-----//
        .on('click', function(event, d) {

          const bounds = pathGenerator.bounds(d),
            dx = bounds[1][0] - bounds[0][0],
            dy = bounds[1][1] - bounds[0][1],
            x = (bounds[0][0] + bounds[1][0]) / 2,
            y = (bounds[0][1] + bounds[1][1]) / 2,
            scale = Math.min(8, 0.9 / Math.max(dx / width, dy / height)),
            translate = [width / 2 - scale * x, height / 2 - scale * y];

          svg.transition()
            .duration(750) // zoom in 的速度
            .call(
              zoomBehavior.transform,
              zoomIdentity.translate(translate[0], translate[1]).scale(scale)
            );

          svg.style('filter', 'url(#blur)');
          showResetButton = true;
          
          // 绘制颜色表
          drawLegend();

          // 调用函数绘制饼状图
          drawPieChart(d.properties.name);
          drawBarChart(d.properties.name);

          // 显示信息圆圈并设置默认文本
          d3.select("#pie-info")
          .style("display", "flex")
        });
    });
  });
  
//-----------------------各个function的区域----------------------//

  // 重置缩放
  function resetZoom() {

    // 移除饼状图容器中的内容
    select("#pie-chart-container").select("svg").remove();
    // Remove bar chart container contents
    d3.select("#bar-chart-container").selectAll("*").remove();

    const svg = select('svg');
    svg.transition()
      .duration(750) // zoom out 的速度
      .call(zoomBehavior.transform, zoomIdentity); // 直接使用zoomIdentity作为重置状态

    svg.style('filter', ''); // 移除毛玻璃效果
    showResetButton = false; // 隐藏重置按钮
    hideLegend() //隐藏图例
    d3.select("#pie-info").style("display", "none"); // 隐藏饼状图的信息圆
  }

  // 绘制饼状图的function
  function drawPieChart(countryName) {
    const data = countryEnergyData[countryName];
    if (!data) return; // 如果该国家没有数据，则直接返回

    // 检查是否所有数据均为0
    const totalValue = Object.values(data).reduce((acc, value) => acc + value, 0);
    const pieData = totalValue === 0
      ? [{name: "No Data", value: 1}] // 如果所有数据均为0，则创建一个代表无数据的数据项
      : Object.entries(data).map(([key, value]) => ({ name: key, value }));

    // 清除之前的饼状图
    d3.select("#pie-chart-container").select("svg").remove();
    // Remove bar chart container contents
    d3.select("#bar-chart-container").selectAll("*").remove();

    const pie = d3.pie().value(d => d.value)(pieData);
    const arc = d3.arc().innerRadius(110).outerRadius(200); // 调整饼状图的大小

    // 创建新的SVG于HTML容器中
    const svg = d3.select("#pie-chart-container").append("svg")
      .attr('width', '100vw') // 使用视口宽度单位
      .attr('height', '100vh') // 使用视口高度单位
      .append('g')
      // 将饼状图中心移动到SVG的中心
      .attr("transform", `translate(${window.innerWidth * 3/4}, ${window.innerHeight * 0.45})`);
    
    // 添加标题
    svg.append("text")
      .attr("x", 0)
      .attr("y", -220) // 根据需要调整
      .attr("text-anchor", "middle")
      .style("font-size", "18px")
      .style("font-weight", "bold")
      .text("Composition of Energy Usage in " + countryName);
    
    svg.selectAll('path')
      .data(pie)
      .join('path')
        .attr('d', arc)
        .attr('fill', d => { 
          if (d.data.name === "No Data") {
            return "#5D6D7E"; // 所有数据均为0时的颜色
          }
          // 根据能源类型分配颜色
          switch(d.data.name) {
            case 'Biofuel': return '#45B39D'; // bio蓝绿色
            case 'Solar': return '#F5B041'; // solar橙黄色
            case 'Hydro': return '#82E0AA'; // hydro浅绿色
            case 'Wind': return '#A9CCE3'; // wind湛蓝色
            case 'Coal': return '#7B241C'; // coal红褐色
            case 'Gas': return '#7D6608'; // gas黄褐色
            case 'Oil': return '#17202A'; // oil黑灰色
            case 'Nuclear': return '#A569BD'; // nuclear紫色
            default: return '#dddddd';
          }
        })
        .attr('stroke', 'white')
        .attr('stroke-width', '2px')

        // 鼠标悬停在pie chart时的行为
        .on('mouseover', function(event, d) {
          // 计算百分比
          const percentage = ((d.data.value / totalValue) * 100).toFixed(1) + "%";
          // 检查数据是否表示"No Data"
          if (d.data.name === "No Data") {
              d3.select("#pie-info").html("No Data");
          } else {
              const consumptionType = d.data.name;
              const consumptionValue = d.data.value;
              d3.select("#pie-info").html(`${consumptionType}: ${consumptionValue} TWh (${percentage})`);
          }
          // 设置选中当前饼状图片段的颜色
          d3.select(this).attr('fill', '#283747');
        })
        .on('mouseout', function() {
          // 鼠标移出时隐藏信息
          d3.select("#pie-info").html("Hover the pie chart to learn more");
          d3.select(this).attr('fill', d => {
              // 根据能源类型重新分配原始颜色
              switch(d.data.name) {
                case 'Biofuel': return '#45B39D'; // bio蓝绿色
                case 'Solar': return '#F5B041'; // solar橙黄色
                case 'Hydro': return '#82E0AA'; // hydro浅绿色
                case 'Wind': return '#A9CCE3'; // wind湛蓝色
                case 'Coal': return '#7B241C'; // coal红褐色
                case 'Gas': return '#7D6608'; // gas黄褐色
                case 'Oil': return '#17202A'; // oil黑灰色
                case 'Nuclear': return '#A569BD'; // nuclear紫色
                default: return '#5D6D7E';
              }
          });
        })
  }

  // 创建rank chart的function
  function drawBarChart(clickedCountry) {
    // Prepare the data
    const sortedCountries = Object.entries(countryEmissions)
      .sort((a, b) => b[1] - a[1]);

    const clickedCountryIndex = sortedCountries.findIndex(([country]) => country === clickedCountry);
    const surroundingCountries = sortedCountries.filter((_, i) => {
      return i <= clickedCountryIndex + 5 && i >= clickedCountryIndex - 5;
    });

    // Clear any existing bar chart
    const barChartContainer = select('#bar-chart-container');
    barChartContainer.selectAll('*').remove();

    // 占用左半边的屏幕
    const screenWidth = window.innerWidth;
    const newWidth = screenWidth / 2;
    const screenHeight = innerHeight;

    // Adjust margins if necessary
    const margin = { top: 60, right: 40, bottom: 30, left: 150 };
    const width = newWidth - margin.left - margin.right;
    const height = screenHeight * 0.61;
    
    const svg = barChartContainer
      .append('svg')
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    // Create scales
    const y = d3.scaleBand()
      .range([0, height])
      .domain(surroundingCountries.map(([country]) => country))
      .padding(0.1);

    const x = d3.scaleLinear()
      .domain([0, d3.max(surroundingCountries, ([, emissions]) => emissions)])
      .range([0, width]);

    // Draw the bars
    svg.selectAll('myRect')
      .data(surroundingCountries)
      .enter()
      .append('rect')
      .attr('x', 0)
      .attr('y', d => y(d[0]))
      .attr('width', d => x(d[1]))
      .attr('height', y.bandwidth())
      .attr('fill', d => d[0] === clickedCountry ? '#C0392B' : '#CA6F1E');//bar chart颜色
    
    svg.selectAll('myText')
      .data(surroundingCountries)
      .enter()
      .append('text')
      .attr('x', d => x(d[1]) + 3) // Move the text a little to the right of the bar's end
      .attr('y', d => y(d[0]) + y.bandwidth() / 2) // Vertically center the text in the bar
      .attr('dy', '.35em') // Adjust vertical alignment
      .text(d => d[1]) // Set the text to the data value
      .attr('fill', 'black') // Set the text color
      .attr('font-size', '10px') // Set the size of the text
      .attr('text-anchor', 'start'); // Anchor the text at the start to ensure it doesn't go past the end of the bar

    // Add axes and labels
    svg.append('g')
      .call(d3.axisLeft(y));
    
    // Add the x-axis
    svg.append('g')
      .attr('transform', `translate(0,${height})`)
      .call(d3.axisBottom(x));
    
    // 添加单位在x轴后
    svg.append("text")
      .attr("transform", `translate(${width}, ${height + margin.bottom - 40})`) // 微调位置参数
      .style("text-anchor", "end")
      .text("MtCO2e");

    // 添加title
    svg.append("text")
      .attr("x", width / 2)
      .attr("y", 0 - (margin.top / 2))
      .attr("text-anchor", "middle") 
      .style("font-size", "18px")
      .style("font-weight", "bold") // 加粗字体
      .text("GHG Emission Rankings: Focus on Neighbors"); // 标题名称
  }

  // 弹出提示的点击操作
  function toggleInfo() {
    showInfo = !showInfo;
  }

  // 弹出数据描述和来源的操作
  function toggleGlobeInfo() {
    showGlobeInfo = !showGlobeInfo;
  }

  // 绘制颜色图例的部分
  const energyTypes = {
    'Biofuel': '#45B39D',
    'Solar': '#F5B041',
    'Hydro': '#82E0AA',
    'Wind': '#A9CCE3',
    'Coal': '#7B241C',
    'Gas': '#7D6608',
    'Oil': '#17202A',
    'Nuclear': '#A569BD',
  };

  function drawLegend() {
    const legendContainer = document.getElementById('color-legend');
    // 清空旧的图例项
    legendContainer.innerHTML = '';

    // 设置为flex以显示图例
    legendContainer.style.display = 'flex';

    Object.entries(energyTypes).forEach(([type, color]) => {
      // 创建颜色方块
      const colorBox = document.createElement('div');
      colorBox.style.width = '12px';
      colorBox.style.height = '12px';
      colorBox.style.backgroundColor = color;
      colorBox.style.marginRight = '3px'; // 添加间隔

      // 创建文本标签
      const textLabel = document.createElement('span');
      textLabel.textContent = type;

      // 创建包含颜色方块和文本的容器
      const legendItem = document.createElement('div');
      legendItem.style.display = 'flex';
      legendItem.style.alignItems = 'center';
      legendItem.style.marginRight = '10px'; // 图例项间的间隔

      // 将颜色方块和文本标签添加到图例项
      legendItem.appendChild(colorBox);
      legendItem.appendChild(textLabel);

      // 将图例项添加到图例容器
      legendContainer.appendChild(legendItem);
    });
  }

  // 隐藏图例的函数
  function hideLegend() {
    document.getElementById('color-legend').style.display = 'none';
  }

  // 跳转链接
  function goToLink() {
    window.open('https://github.com/KUJIcheng/GHG-Emission-Map/blob/main/write-up.md', '_blank');
  }
</script>

<!-- 提示icon 图层为3-->
<img src="icon/icons8-idea-sharing-64.png" id="help-icon" alt="Help Icon" style="cursor: pointer; position: fixed; left: 23px; bottom: 23px; width: 48px; height: 48px; z-index: 3;" on:click={toggleInfo}>

<!-- 地球icon 图层为3-->
<img src="icon/icons8-earth-globe-64.png" id="globe-icon" alt="Globe Icon" 
     style="cursor: pointer; position: fixed; right: 20px; top: 20px; width: 48px; height: 48px; z-index: 3;" 
     on:click={toggleGlobeInfo}>

<!-- writeup icon 图层为3 -->
<img src="icon/icons8-training-64.png" alt="Training Icon" 
  style="cursor: pointer; position: fixed; left: 20px; top: 20px; width: 48px; height: 48px; z-index: 3;" 
  on:click={goToLink}>

<!-- 在writeup icon下方添加一行文字 -->
<span style="position: fixed; left: -2px; top: 68px; z-index: 3; color: black; background-color: rgba(255,255,255,0.0); padding: 2px 8px; border-radius: 4px; font-size: 12px;">
  Project Writeup
</span>

<!-- 在地球icon下方添加一行文字 -->
<span style="position: fixed; right: -4px; top: 68px; z-index: 3; color: black; background-color: rgba(255,255,255,0.0); padding: 2px 8px; border-radius: 4px; font-size: 12px;">
  Data Description
</span>

<!-- 数据描述文本 图层为3-->
{#if showGlobeInfo}
<div id="globe-info-layer" style="position: fixed; right: 20px; top: 70px; background: rgba(255, 255, 255, 0.8); border-radius: 10px; padding: 10px; width: 300px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); z-index: 3;">
  Explore Emissions and Energy Consumption Connections: Dive into the relation and association between national greenhouse gas outputs and energy consumption profiles.🤔
  <br><br>
  The dataset sourced from <a href="https://github.com/owid" target="_blank" rel="noopener noreferrer">Our World in Data</a> 🌍 presents comprehensive global energy data including consumption, production, and the variety of energy sources, as well as greenhouse gas (GHG) emissions. 
  <br><br>
  This map focuses on the GHG emissions data and national energy compositions of 2018, allowing users to interpret GHG emissions and their energy breakdown.🗺️
</div>
{/if}

<!-- 提示文本框 图层为3-->
{#if showInfo}
<div id="info-layer" style="position: fixed; left: 20px; bottom: 70px; background: rgba(255, 255, 255, 0.8); border-radius: 10px; padding: 10px; width: 300px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); z-index: 3;" >
  Interaction Tips:✨
  <br><br>
  Use the mouse wheel to zoom in to select a country and obtain the country's greenhouse gas emissions by hovering.🕹️
  <br><br>
  Click on the selected country to get more data about the country, including the country's energy consumption and proportion, as well as the ranking of greenhouse gas emissions compared to other countries.🖱️
</div>
{/if}

<!-- 鼠标悬停时显示国家的排放数值 图层为1-->
<div id="tooltip" style="position: absolute; opacity: 0; background: #fff; border: 0px solid #000; padding: 5px; pointer-events: none; z-index: 1;"></div>

<!-- 饼状图层级 图层为2-->
<div id="pie-chart-container" style="position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 2;"></div>
<!-- rank chart图层级 图层为2-->
<div id="bar-chart-container" style="position: fixed; left: 0%; top: 10%; z-index: 2;"></div>

<!-- 悬停饼状图信息框 图层为3 默认隐藏-->
<div id="pie-info" style="text-align: center; position: fixed; left: 75%; top: 45%; transform: translate(-50%, -50%); background: rgba(255, 255, 255, 0.6); padding: 10px; border-radius: 50%; width: 150px; height: 150px; display: none; align-items: center; justify-content: center; z-index: 3;">
  Hover the pie chart to learn more
</div>

<!-- 颜色图例的图层 图层为3 默认隐藏-->
<div id="color-legend" style="display: none; position: fixed; top: 79%; left: 77%; transform: translateX(-50%); z-index: 3; flex-direction: row;"></div>

<svg>
  <defs>
    <filter id="blur">
      <feGaussianBlur in="SourceGraphic" stdDeviation="5"></feGaussianBlur>
    </filter>
  </defs>
</svg>

<!-- 添加标题 -->
<h1 class="app-title">Interactive GHG Emissions Map with National Energy Profiles</h1>

{#if showResetButton}
  <button on:click={resetZoom} class="reset-button">Return to World Map</button>
{/if}

<style>
  svg {
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #D4E6F1;
  }
  
  :global(svg path) {
    stroke: rgb(102, 102, 102);
    stroke-width: 0.3;
  }

  .reset-button {
    position: absolute;
    bottom: 4%;
    left: 50%;
    transform: translateX(-50%);
    padding: 10px 20px;
    border-radius: 20px;
    border: none;
    background-color: rgba(255, 255, 255, 0.8);
    cursor: pointer;
    transition: background-color 0.3s;
    z-index: 3;
  }

  .reset-button:hover {
    background-color: rgba(8, 172, 172, 0.6);
  }

  /* 标题样式 */
  .app-title {
    position: absolute;
    top: -1.6%; /* 距离页面顶部的位置 */
    left: 50%;
    transform: translateX(-50%); /* 水平居中 */
    font-size: 125%; /* 字体大小 */
    font-weight: bold; /* 字体粗细 */
    color: #333; /* 字体颜色 */
    text-align: center; /* 文字对齐方式 */
  }
</style>