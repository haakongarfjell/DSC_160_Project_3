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
    let tooltip;


  
    onMount(async () => {
        svg = d3.select("#my_dataviz");
        width = window.innerWidth;
        height = window.innerHeight * 0.9;
    
        projection = d3.geoMercator()
            .scale(150)
            .translate([width / 2, height / 1.5]);
    
        path = d3.geoPath().projection(projection);
    
        colorScale = d3.scaleSequential()
          .domain([0, 1]) 
          .interpolator(d3.interpolateBlues);
    
        tooltip = d3.select("body").append("div")
            .attr("class", "tooltip")
            .style("opacity", 0)
            .style("position", "absolute")
            .style("top", "300px") 
            .style("left", "20px");
    
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
        console.log(minTemp);
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

        const title = document.createElement('h1');
        title.textContent = 'Where do you want to live?';
        title.style.position = 'absolute';
        title.style.top = '0%';
        title.style.left = '50%';
        title.style.transform = 'translate(-50%, -50%)'; 
        document.body.insertBefore(title, document.body.firstChild);

        const subtitle = document.createElement('h2');
        subtitle.textContent = 'Adjust the sliders to find the best country for you!';
        subtitle.style.position = 'absolute';
        subtitle.style.top = '50px'; 
        subtitle.style.left = '50%';
        subtitle.style.transform = 'translateX(-50%)'; 
        document.body.insertBefore(subtitle, document.body.firstChild);

    });

    async function loadTemperatureData(data) {
      if (!data || !Array.isArray(data) || data.length === 0) {
        console.error("Data is invalid or empty.");
        return null;
      }

      const groupedData = data.reduce((acc, currentValue) => {
          const country = currentValue.Country;
          const avgYear = +currentValue.Avg_Year;

          if (!acc.has(country)) {
              acc.set(country, []);
          }

          acc.get(country).push(avgYear);

          return acc;
      }, new Map());

      const selectedData = new Map();
      groupedData.forEach((temperatures, country) => {
          const avgTemperature = temperatures.reduce((total, temp) => total + temp, 0) / temperatures.length;
          selectedData.set(country, avgTemperature);
      });

      return selectedData;
    }

    function updateMap(topo) {
        svg.selectAll(".country").remove();

        let topCountries = [];

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

            topCountries.push({ 
                name: d.properties.name, 
                weightedValue: weightedValue,
                hdi: hdi,
                temperature: temperature,
                population: population
            });

            return colorScale(weightedValue);
            })
            .on("mouseover", function (d) {
              d3.select(this).style("fill", function (d) {
              const population = populationData.get(d.id) || 0;
              const hdi = HDIdata.get(d.id) || 0;
              const temperature = temperatureData.get(d.properties.name) || 0;
              d3.select(this).style("fill", "#ffab00");
              tooltip.transition()
                  .duration(200)
                  .style("opacity", .9);
              tooltip.html(
                  `Country: ${d.properties.name}<br>
                  HDI: ${hdi}<br>
                  Avg Temp: ${temperature.toFixed(2)} C<br>
                  Population: ${population.toLocaleString()}`
              )
                  .style("left", (d3.event.pageX + 10) + "px")
                  .style("top", (d3.event.pageY - 28) + "px");
                });
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

          topCountries.sort((a, b) => b.weightedValue - a.weightedValue);

          const top5Countries = topCountries.slice(0, 5);

          const highestWeightedCountryDiv = document.getElementById("highestWeightedCountry");
          highestWeightedCountryDiv.innerHTML = "<h3>Top 5 candidate countries:</h3>";
          top5Countries.forEach(country => {
              const roundedTemperature = country.temperature.toFixed(2);
              highestWeightedCountryDiv.innerHTML += `<p>${country.name}, HDI: ${country.hdi}, Avg. Temp: ${roundedTemperature} C, Population: ${country.population.toLocaleString()}</p>`;
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
        left: 7.5%;
        bottom: 20px;
        width: calc(100% - 40px);
      }
       .slider-container_second {
        position: absolute;
        left: 37.5%;
        bottom: 20px;
        width: calc(100% - 40px);
      }
        .slider-container_third {
        position: absolute;
        left: 67.5%;
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
        width: 100%;
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
          top: 50px; 
          left: 20px; 
      }

      .highest-weighted-country {
        position: absolute;
        top: 20px; 
        left: 20px; 
        background-color: white; 
        padding: 10px; 
        border: 1px solid black; 
        border-radius: 5px; 
      }
    </style>
    
    <svg id="my_dataviz"></svg>
    
    <div class="slider-container" style="width: 25%;">
      <p>Population:</p>
      <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="weightSlider">
    </div>
    
    <div class="slider-container_second" style="width: 25%;">
      <p>HDI:</p>
      <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="secondDataSlider">
    </div>
    
    <div class="slider-container_third" style="width: 25%;">
      <p>Average temperature:</p>
      <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="thirdDataSlider">
    </div>

  <div id="highestWeightedCountry" class="highest-weighted-country"></div>
