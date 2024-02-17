<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

  
    let svg;
    let width;
    let height;
    let projection;
    let path;
    let data = new Map();
    let secondData = new Map();
    let colorScale;
    let weightPop = 0.5;
    let weightHDI = 0.5;
    let weightTemp = 0.5;

    let populationData;
    let HDIdata;
    let temperatureData;

    let minHDI;
    let maxHDI;

    let minPop;
    let maxPop;

    let minTemp;
    let maxTemp;


  
    onMount(async () => {
        svg = d3.select("#my_dataviz");
        width = window.innerWidth;
        height = window.innerHeight * 0.9;
    
        projection = d3.geoMercator()
            .scale(150)
            .translate([width / 2, height / 1.5]);
    
        path = d3.geoPath().projection(projection);
    
        colorScale = d3.scaleSequential()
          .domain([0, 1]) // Define the domain as the range of values your data can take
          .interpolator(d3.interpolateBlues);
    
        const tooltip = d3.select("body").append("div")
            .attr("class", "tooltip")
            .style("opacity", 0);
    
        const [topoData, popData, HDIdataCsv, tempData ] = await Promise.all([
            d3.json("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson"),
            d3.csv("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world_population.csv"),
            d3.csv("HDIdata.csv"),
            d3.csv("Avg_World_Temp_2020.csv")
        ]);
    
        populationData = new Map(popData.map(d => [d.code, +d.pop]));
        HDIdata = new Map(HDIdataCsv.map(d => [d.code, +d.hdi]));
        temperatureData = await loadTemperatureData(tempData);
    

        minHDI = d3.min(HDIdataCsv, d => +d.hdi) + 0.01;
        maxHDI = d3.max(HDIdataCsv, d => +d.hdi);

        minPop = d3.min(popData, d => +d.pop);
        maxPop = d3.max(popData, d => +d.pop);

        minTemp = d3.min(tempData, d => +d.Avg_Year);
        maxTemp = d3.max(tempData, d => +d.Avg_Year);

        
        svg.attr("width", width)
            .attr("height", height);
    
        updateMap(topoData);
    
        d3.select("#weightSlider").on("input", function () {
            weightPop = +this.value;
            updateMap(topoData);
        });
    
        d3.select("#secondDataSlider").on("input", function () {
            weightHDI = +this.value;
            updateMap(topoData);
        });

        d3.select("#thirdDataSlider").on("input", function () {
            weightTemp = +this.value;
            updateMap(topoData);
        });
    });

    async function loadTemperatureData(data) {
      if (!data || !Array.isArray(data) || data.length === 0) {
        console.error("Data is invalid or empty.");
        return null;
      }

      // Group data by country
      const groupedData = data.reduce((acc, currentValue) => {
          const country = currentValue.Country;
          const avgYear = +currentValue.Avg_Year;

          // If the country is not in the accumulator yet, add it
          if (!acc.has(country)) {
              acc.set(country, []);
          }

          // Push the average temperature to the array for the country
          acc.get(country).push(avgYear);

          return acc;
      }, new Map());

      // Calculate the average temperature for each country
      const selectedData = new Map();
      groupedData.forEach((temperatures, country) => {
          const avgTemperature = temperatures.reduce((total, temp) => total + temp, 0) / temperatures.length;
          selectedData.set(country, avgTemperature);
      });

      return selectedData;
    }

    function updateMap(topo) {
        svg.selectAll(".country").remove();

        svg.selectAll(".country")
            .data(topo.features)
            .enter().append("path")
            .attr("class", "country")
            .attr("d", path)
            .style("fill", function (d) {
            const population = populationData.get(d.id) || 0;
            const hdi = HDIdata.get(d.id) || 0;
            const temperature = temperatureData.get(d.properties.name) || 0;

            const normalizedPopulation = (population - minPop) / (maxPop - minPop);
            const normalizedHDI = hdi === 0 ? 0 : (hdi - minHDI) / (maxHDI - minHDI);
            const normalizedTemperature = (temperature - minTemp) / (maxTemp - minTemp);

            const weightedValue = (weightPop - 0.33*weightHDI - 0.33*weightTemp) * normalizedPopulation + (weightHDI - 0.33*weightPop - 0.33*weightTemp) * normalizedHDI + (weightTemp - 0.33*weightPop - 0.33*weightHDI) * normalizedTemperature;

            return colorScale(weightedValue);
            })
            .on("mouseover", function (d) {
            const population = populationData.get(d.id) || 0;
            d3.select(this).style("fill", "#ffab00");
            tooltip.transition()
                .duration(200)
                .style("opacity", .9);
            tooltip.html("Country: " + d.properties.name + "<br>Population: " + population.toLocaleString())
                .style("left", (d3.event.pageX + 10) + "px")
                .style("top", (d3.event.pageY - 28) + "px");
            })
            .on("mouseout", function (d) {
            d3.select(this).style("fill", function (d) {
                const population = populationData.get(d.id) || 0;
                const hdi = HDIdata.get(d.id) || 0;
                const temperature = temperatureData.get(d.properties.name) || 0;

                const normalizedPopulation = (population - minPop) / (maxPop - minPop);
                const normalizedHDI = hdi === 0 ? 0 : (hdi - minHDI) / (maxHDI - minHDI);
                const normalizedTemperature = (temperature - minTemp) / (maxTemp - minTemp);

                const weightedValue = (weightPop - 0.33*weightHDI - 0.33*weightTemp) * normalizedPopulation + (weightHDI - 0.33*weightPop - 0.33*weightTemp) * normalizedHDI + (weightTemp - 0.33*weightPop - 0.33*weightHDI) * normalizedTemperature;
                return colorScale(weightedValue);
            });
            tooltip.transition()
                .duration(500)
                .style("opacity", 0);
            });
        }

  </script>
  
  
    <style>
      body, html {
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
      }
    
      svg {
        display: block;
        width: 100%;
        height: 90%;
      }
    
      .slider-container {
        position: absolute;
        left: 4%;
        bottom: 20px;
        width: calc(100% - 40px);
      }
       .slider-container_second {
        position: absolute;
        left: 28%;
        bottom: 20px;
        width: calc(100% - 40px);
      }
        .slider-container_third {
        position: absolute;
        left: 52%;
        bottom: 20px;
        width: calc(100% - 40px);
      }
        .slider-container_fourth {
        position: absolute;
        left: 76%;
        bottom: 20px;
        width: calc(100% - 40px);
      }
    
      .slider {
        width: 20%;
      }
    
      .country {
        fill: #e0e0e0;
        stroke: #fff;
        stroke-width: 0.5px;
      }
    
      .country:hover {
        fill: #bfbfbf;
      }
    
      .tooltip {
        position: absolute;
        padding: 10px;
        background: #fff;
        border: 1px solid #ccc;
        border-radius: 5px;
        pointer-events: none;
        opacity: 0;
        transition: opacity 0.3s;
      }
    </style>
    
    <svg id="my_dataviz"></svg>
    
  <div class="slider-container">
    <p>Weighted Population:</p>
    <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="weightSlider">
  </div>
  
  <div class="slider-container_second">
      <p>Second Data Weight:</p>
      <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="secondDataSlider">
  </div>
  
  <div class="slider-container_third">
      <p>Third Data Weight:</p>
      <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="thirdDataSlider">
  </div>
  
  <div class="slider-container_fourth">
      <p>Fourth Data Weight:</p>
      <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="ForthDataSlider">
  </div>