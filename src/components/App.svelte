<script>
  import { onMount } from 'svelte';
  import { select, geoPath, geoNaturalEarth1, json, zoom, zoomIdentity, csv, scaleSequential, interpolateRgb} from 'd3';
  import { feature } from 'topojson-client';
  import * as d3 from 'd3';

  let showResetButton = false;
  let zoomBehavior;
  let countryEmissions = {};
  let colorScale;

  // pie chart data storage
  let countryEnergyData = {};
  // Default state of the tips text box
  let showInfo = false;
  // Default state of the data description text box
  let showGlobeInfo = false;
  
  let maxEmission = 0;

  //-----Code storage area when the project is running, area for reading csv and json-----//
  onMount(() => {
    const width = window.innerWidth;
    const height = window.innerHeight;
    // Calculate the zoom of the world map using the appropriate scale
    const scale = Math.min(width, height) / 2.95;
    const svg = select('svg');
    const projection = geoNaturalEarth1().scale(scale).translate([width / 2 - 18, height / 2 + 15]);
    const pathGenerator = geoPath().projection(projection);

    const tooltip = d3.select("#tooltip"); // create tooltip

    zoomBehavior = zoom().scaleExtent([1, 8]).translateExtent([[0, 0], [width, height]])
      .on('zoom', (event) => {
        svg.selectAll('path').attr('transform', event.transform);
      });

    svg.call(zoomBehavior);

    // Loading CSV data
    csv('energy_consumption_2018.csv').then(data => {
      maxEmission = data.reduce((max, d) => Math.max(max, +d.greenhouse_gas_emissions), 0);
      // Setting the color scale
      colorScale = scaleSequential([0, maxEmission], t => interpolateRgb("#FDEBD0", "#A04000")(t));

      data.forEach(d => {
        // Storing GHG emission data
        countryEmissions[d.country] = +d.greenhouse_gas_emissions;

        // Storing energy consumption data
        countryEnergyData[d.country] = {
          //Renewable class:
          Biofuel: +d.biofuel_consumption,
          Solar: +d.solar_consumption,
          Hydro: +d.hydro_consumption,
          Wind: +d.wind_consumption,
          //Fossil class:
          Coal: +d.coal_consumption,
          Gas: +d.gas_consumption,
          Oil: +d.oil_consumption,
          //low carbon class:
          Nuclear: +d.nuclear_consumption,
        };
      });drawMapLegend()
    });

    // Loading Json data
    json('countries-110m.json').then(data => {
      const countries = feature(data, data.objects.countries);
      svg.selectAll('path')
        .data(countries.features)
        .join('path')
        .attr('d', pathGenerator)
        .attr('fill', d => countryEmissions[d.properties.name] ? colorScale(countryEmissions[d.properties.name]) : '#D5DBDB') // 默认颜色
        
        //-----Mouse hover behavior-----//
        .on('mouseover', function(event, d) {
          select(this).style('fill', '#566573');
          tooltip.style("opacity", 0.8)
                .html(d.properties.name + "<br/>Greenhouse Gas Emissions: " + (countryEmissions[d.properties.name] + " MtCO2e" || "N/A"))
                .style("left", (event.pageX + 20) + "px")
                .style("top", (event.pageY - 51) + "px")
                .style("border-radius", "10px");
        })
        //-----Mouse moving behavior-----//
        .on('mousemove', function(event) {
          tooltip.style("left", (event.pageX + 20) + "px") // The message distance from mouse
                .style("top", (event.pageY - 51) + "px");
        })
        //-----Mouse leave behavior-----//
        .on('mouseout', function(event, d) {
          select(this).style('fill', countryEmissions[d.properties.name] ? colorScale(countryEmissions[d.properties.name]) : '#D5DBDB');
          tooltip.style("opacity", 0);
        })

        //-----Mouse click logic-----//
        .on('click', function(event, d) {

          const bounds = pathGenerator.bounds(d),
            dx = bounds[1][0] - bounds[0][0],
            dy = bounds[1][1] - bounds[0][1],
            x = (bounds[0][0] + bounds[1][0]) / 2,
            y = (bounds[0][1] + bounds[1][1]) / 2,
            scale = Math.min(8, 0.9 / Math.max(dx / width, dy / height)),
            translate = [width / 2 - scale * x, height / 2 - scale * y];

          svg.transition()
            .duration(750) // Speed of zoom in
            .call(
              zoomBehavior.transform,
              zoomIdentity.translate(translate[0], translate[1]).scale(scale)
            );
          
          // show the reset bottom and activate the blur effect
          svg.style('filter', 'url(#blur)');
          showResetButton = true;
          
          // Creating color legend
          drawLegend();

          // Call function to draw pie chart and bar chart
          drawPieChart(d.properties.name);
          drawBarChart(d.properties.name);

          // Show the message circle in pie chart
          d3.select("#pie-info")
          .style("display", "flex")

          // Hiding the color legend
          d3.select("#legend-container").style("display", "none");
        });
    });
  });
  
//-----------------------Function start here----------------------//

  // rest the zoom status
  function resetZoom() {

    // 移除饼状图容器中的内容
    select("#pie-chart-container").select("svg").remove();
    // Remove bar chart container contents
    d3.select("#bar-chart-container").selectAll("*").remove();

    const svg = select('svg');
    svg.transition()
      .duration(750) // speed of zoom out
      .call(zoomBehavior.transform, zoomIdentity); // using zoomIdentity as default status
    svg.style('filter', ''); // remove the blur effect
    showResetButton = false; // hiding the reset bottom
    hideLegend() // Hiding the color legend
    d3.select("#pie-info").style("display", "none"); // Hiding the message circle in the pie chart
    d3.select("#legend-container").style("display", "block");
  }

  // The function to create the pie chart
  function drawPieChart(countryName) {
    const data = countryEnergyData[countryName];
    if (!data) return;
    // Checking if all the data is 0
    const totalValue = Object.values(data).reduce((acc, value) => acc + value, 0);
    const pieData = totalValue === 0
      ? [{name: "No Data", value: 1}] // If all data is 0, create a data item representing no data
      : Object.entries(data).map(([key, value]) => ({ name: key, value }));

    // Remove the pie chart
    d3.select("#pie-chart-container").select("svg").remove();
    // Remove bar chart container contents
    d3.select("#bar-chart-container").selectAll("*").remove();

    const pie = d3.pie().value(d => d.value)(pieData);
    const arc = d3.arc().innerRadius(110).outerRadius(200);

    // Create SVG into the HTML layer
    const svg = d3.select("#pie-chart-container").append("svg")
      .attr('width', '100vw')
      .attr('height', '100vh')
      .append('g')
      // morve the pie chart to the SVG center
      .attr("transform", `translate(${window.innerWidth * 3/4}, ${window.innerHeight * 0.45})`);

    svg.append("text")
      .attr("x", 0)
      .attr("y", -220)
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
            return "#5D6D7E"; // Default color when all data is 0
          }
          // Assign colors based on energy type
          switch(d.data.name) {
            case 'Biofuel': return '#45B39D';
            case 'Solar': return '#F5B041';
            case 'Hydro': return '#82E0AA';
            case 'Wind': return '#A9CCE3';
            case 'Coal': return '#7B241C';
            case 'Gas': return '#7D6608';
            case 'Oil': return '#17202A';
            case 'Nuclear': return '#A569BD';
            default: return '#dddddd';
          }
        })
        .attr('stroke', 'white')
        .attr('stroke-width', '2px')

        // When mouse hover on the pie chart
        .on('mouseover', function(event, d) {
          // Calculate the percentage
          const percentage = ((d.data.value / totalValue) * 100).toFixed(1) + "%";
          // Check if the object has "No Data"
          if (d.data.name === "No Data") {
              d3.select("#pie-info").html("No Data");
          } else {
              const consumptionType = d.data.name;
              const consumptionValue = d.data.value;
              d3.select("#pie-info").html(`${consumptionType}: ${consumptionValue} TWh (${percentage})`);
          }
          // Setting the color when the pie chart part is havor
          d3.select(this).attr('fill', '#283747');
        })
        .on('mouseout', function() {
          // Hiding the message when the mouse leave
          d3.select("#pie-info").html("Hover the pie chart to learn more");
          d3.select(this).attr('fill', d => {
              // Assign colors based on energy type
              switch(d.data.name) {
                case 'Biofuel': return '#45B39D';
                case 'Solar': return '#F5B041';
                case 'Hydro': return '#82E0AA';
                case 'Wind': return '#A9CCE3';
                case 'Coal': return '#7B241C';
                case 'Gas': return '#7D6608';
                case 'Oil': return '#17202A';
                case 'Nuclear': return '#A569BD';
                default: return '#5D6D7E';
              }
          });
        })
  }

  // Function of creating bar chart
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

    // Place on the left of the screen
    const screenWidth = window.innerWidth;
    const newWidth = screenWidth / 2;
    const screenHeight = innerHeight;

    const margin = { top: 60, right: 40, bottom: 30, left: 150 };
    const width = newWidth - margin.left - margin.right;
    const height = screenHeight * 0.61;
    
    const svg = d3.select('#bar-chart-container')
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
    const bars = svg.selectAll('rect')
      .data(surroundingCountries)
      .enter()
      .append('rect')
      .attr('x', 0)
      .attr('y', d => y(d[0]))
      .attr('width', d => x(d[1]))
      .attr('height', y.bandwidth())
      .attr('fill', d => d[0] === clickedCountry ? '#C0392B' : '#CA6F1E');
    
    svg.selectAll('myText')
      .data(surroundingCountries)
      .enter()
      .append('text')
      .attr('x', d => x(d[1]) + 3)
      .attr('y', d => y(d[0]) + y.bandwidth() / 2)
      .attr('dy', '.35em')
      .text(d => d[1])
      .attr('fill', 'black')
      .attr('font-size', '10px')
      .attr('text-anchor', 'start');

    // Add axes and labels
    svg.append('g')
      .call(d3.axisLeft(y));
    
    // Add the x-axis
    svg.append('g')
      .attr('transform', `translate(0,${height})`)
      .call(d3.axisBottom(x));
    
    // Add units after x-axis
    svg.append("text")
      .attr("transform", `translate(${width}, ${height + margin.bottom - 40})`)
      .style("text-anchor", "end")
      .text("MtCO2e");

    // Adding title to the bar chart
    svg.append("text")
      .attr("x", width / 2)
      .attr("y", 0 - (margin.top / 2))
      .attr("text-anchor", "middle") 
      .style("font-size", "18px")
      .style("font-weight", "bold")
      .text("GHG Emission Rankings: Focus on Neighbors");
  }

  // Function to call the tips text box
  function toggleInfo() {
    showInfo = !showInfo;
  }

  // Function to call the data description text box
  function toggleGlobeInfo() {
    showGlobeInfo = !showGlobeInfo;
  }

  // The default color setting to different type of energy consumption
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
    // Clear the color legend of pie chart
    legendContainer.innerHTML = '';

    // Set the display as flex to let the legend appear
    legendContainer.style.display = 'flex';

    Object.entries(energyTypes).forEach(([type, color]) => {
      // Creating the color box
      const colorBox = document.createElement('div');
      colorBox.style.width = '12px';
      colorBox.style.height = '12px';
      colorBox.style.backgroundColor = color;
      colorBox.style.marginRight = '3px';

      // Creating label to different color box
      const textLabel = document.createElement('span');
      textLabel.textContent = type;

      // Create a container containing color squares and text
      const legendItem = document.createElement('div');
      legendItem.style.display = 'flex';
      legendItem.style.alignItems = 'center';
      legendItem.style.marginRight = '10px';

      // Add color squares and text labels to legend items
      legendItem.appendChild(colorBox);
      legendItem.appendChild(textLabel);

      // Adding the graph into the SVG container
      legendContainer.appendChild(legendItem);
    });
  }

  // Function of building a world map Legend
  function drawMapLegend() {
    const legendContainer = d3.select('#legend-container').style("display", "block");
    const svgHeight = 200;
    const svgWidth = 80;
    const segments = 10;
    const segmentHeight = svgHeight / segments;

    // Remove the old world map Legend
    legendContainer.selectAll("*").remove();

    // Ceacte a SVG container
    const svg = legendContainer.append("svg")
      .attr("width", svgWidth)
      .attr("height", svgHeight)
      .style("position", "fixed")
      .style("left", "25px")
      .style("top", "65%")
      .style("transform", "translateY(-50%)");

    // Create a gradient color segment with values increasing from bottom to top
    for (let i = 0; i < segments; i++) {
      svg.append("rect")
        .attr("x", 0)
        .attr("y", svgHeight - (i + 1) * segmentHeight) // Start from the bottom
        .attr("width", 20)
        .attr("height", segmentHeight)
        .attr("fill", colorScale(maxEmission * (i / (segments - 1)))); // Color changing setting

      // Add label to the color block
      if (i < segments - 1) {
        svg.append("text")
          .attr("x", 25)
          .attr("y", svgHeight - i * segmentHeight - segmentHeight / 2)
          .attr("dy", ".35em")
          .text(`${Math.round(maxEmission * (i / (segments - 1)))} MtCO2e`)
          .attr("font-size", "8px");
      } else {
        svg.append("text")
          .attr("x", 25)
          .attr("y", 10)
          .text(`${Math.round(maxEmission * (i / (segments - 1)))} MtCO2e`)
          .attr("font-size", "8px");
      }
    }
  }

  // Function to hide the Legend
  function hideLegend() {
    document.getElementById('color-legend').style.display = 'none';
  }

  // Link to the write up page
  function goToLink() {
    window.open('https://github.com/KUJIcheng/GHG-Emission-Map/blob/main/write-up.md', '_blank');
  }
</script>

<!-- Tips icon Layer 3-->
<img src="icon/icons8-idea-sharing-64.png" id="help-icon" alt="Help Icon" style="cursor: pointer; position: fixed; left: 23px; bottom: 23px; width: 48px; height: 48px; z-index: 3;" on:click={toggleInfo}>

<!-- Earth icon; Layer 3-->
<img src="icon/icons8-earth-globe-64.png" id="globe-icon" alt="Globe Icon" 
     style="cursor: pointer; position: fixed; right: 20px; top: 20px; width: 48px; height: 48px; z-index: 3;" 
     on:click={toggleGlobeInfo}>

<!-- Writeup icon; Layer 3 -->
<img src="icon/icons8-training-64.png" alt="Training Icon" 
  style="cursor: pointer; position: fixed; left: 20px; top: 20px; width: 48px; height: 48px; z-index: 3;" 
  on:click={goToLink}>

<!-- Add a line of text below the writeup icon -->
<span style="position: fixed; left: -2px; top: 68px; z-index: 3; color: black; background-color: rgba(255,255,255,0.0); padding: 2px 8px; border-radius: 4px; font-size: 12px;">
  Project Writeup
</span>

<!-- Add a line of text below the earth icon -->
<span style="position: fixed; right: -4px; top: 68px; z-index: 3; color: black; background-color: rgba(255,255,255,0.0); padding: 2px 8px; border-radius: 4px; font-size: 12px;">
  Data Description
</span>

<!-- Data Description text box; Layer 4-->
{#if showGlobeInfo}
<div id="globe-info-layer" style="position: fixed; right: 20px; top: 70px; background: rgba(255, 255, 255, 0.9); border-radius: 10px; padding: 10px; width: 300px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); z-index: 4;">
  Explore Emissions and Energy Consumption Connections: Dive into the relation and association between national greenhouse gas outputs and energy consumption profiles.🤔
  <br><br>
  The dataset sourced from <a href="https://github.com/owid" target="_blank" rel="noopener noreferrer">Our World in Data</a> 🌍 presents comprehensive global energy data including consumption, production, and the variety of energy sources, as well as greenhouse gas (GHG) emissions. 
  <br><br>
  This map focuses on the GHG emissions data and national energy compositions of 2018, allowing users to interpret GHG emissions and their energy breakdown.🗺️
  <br><br>
  "TWh" (Terawatt-hour) measures energy use, equivalent to one trillion watts for an hour. "MtCO2e" (Metric tons of CO2 equivalent) quantifies greenhouse gas emissions, converting various gases into their CO2 impact for comparison, facilitating comparison of their global warming potential.🌡️
</div>
{/if}

<!-- Tips text box; Layer 4-->
{#if showInfo}
<div id="info-layer" style="position: fixed; left: 20px; bottom: 70px; background: rgba(255, 255, 255, 0.9); border-radius: 10px; padding: 10px; width: 300px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); z-index: 4;" >
  Interaction Tips:✨
  <br><br>
  Use the mouse wheel to zoom in to select a country and obtain the country's greenhouse gas emissions by hovering.🕹️
  <br><br>
  Click on the selected country to get more information about the country, including the country's energy consumption and proportion, as well as the ranking of greenhouse gas emissions compared to other countries.🖱️
</div>
{/if}

<!-- Displays the country's emission values on mouseover. Layer 1-->
<div id="tooltip" style="position: absolute; opacity: 0; background: #fff; border: 0px solid #000; padding: 5px; pointer-events: none; z-index: 1;"></div>

<!-- Pie chart; Layer 2-->
<div id="pie-chart-container" style="position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 2;"></div>
<!-- Rank chart Layer 2-->
<div id="bar-chart-container" style="position: fixed; left: 0%; top: 10%; z-index: 2;"></div>

<!-- Message circle; Layer3; Default hide-->
<div id="pie-info" style="text-align: center; position: fixed; left: 75%; top: 45%; transform: translate(-50%, -50%); background: rgba(255, 255, 255, 0.6); padding: 10px; border-radius: 50%; width: 150px; height: 150px; display: none; align-items: center; justify-content: center; z-index: 3;">
  Hover the pie chart to learn more
</div>

<!-- Pie chart color legend; Layer3; Default hide-->
<div id="color-legend" style="display: none; position: fixed; top: 79%; left: 77%; transform: translateX(-50%); z-index: 3; flex-direction: row;"></div>

<svg>
  <defs>
    <filter id="blur">
      <feGaussianBlur in="SourceGraphic" stdDeviation="5"></feGaussianBlur>
    </filter>
  </defs>
</svg>

<!-- Color legend container -->
<div id="legend-container" style="position: fixed; left: 20px; top: 20px; z-index: 1;"></div>

<!-- Adding title -->
<h1 class="app-title">Carbon Cartography:<br>Navigating 2018's Energy Consumption and Greenhouse Gas Emissions<br></h1>

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

  /* Title style setting */
  .app-title {
    position: absolute;
    top: -1.6%;
    left: 50%;
    transform: translateX(-50%);
    font-size: 92%;
    font-weight: bold;
    color: #333;
    text-align: center;
  }
</style>