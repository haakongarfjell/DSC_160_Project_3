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
    let populationData;
    let HDIdata;

    let minHDI;
    let maxHDI;

    let minPop;
    let maxPop;


  
    onMount(async () => {
        svg = d3.select("#my_dataviz");
        width = window.innerWidth;
        height = window.innerHeight * 0.9;
    
        projection = d3.geoMercator()
            .scale(150)
            .translate([width / 2, height / 1.5]);
    
        path = d3.geoPath().projection(projection);
    
        colorScale = d3.scaleThreshold()
            .domain([0.3, 0.5, 0.6, 0.7, 0.8, 1.0])
            .range(d3.schemeBlues[7]);
    
        const tooltip = d3.select("body").append("div")
            .attr("class", "tooltip")
            .style("opacity", 0);
    
        const [topoData, popData, HDIdataCsv ] = await Promise.all([
            d3.json("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson"),
            d3.csv("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world_population.csv"),
            d3.csv("HDIdata.csv")
        ]);
    
        populationData = new Map(popData.map(d => [d.code, +d.pop]));
        HDIdata = new Map(HDIdataCsv.map(d => [d.code, +d.hdi]));
    

        minHDI = d3.min(HDIdataCsv, d => +d.hdi) + 0.01;
        maxHDI = d3.max(HDIdataCsv, d => +d.hdi);

        minPop = d3.min(popData, d => +d.pop);
        maxPop = d3.max(popData, d => +d.pop);

        
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
    });
  
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

            const normalizedPopulation = (population - minPop) / (maxPop - minPop);
            const normalizedHDI = (hdi - minHDI) / (maxHDI - minHDI);

            // Apply equal weights to normalized values
            const weightedValue = (weightPop - 0.5*weightHDI) * normalizedPopulation + (weightHDI - 0.5*weightPop) * normalizedHDI;

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

                const normalizedPopulation = (population - minPop) / (maxPop - minPop);
                const normalizedHDI = (hdi - minHDI) / (maxHDI - minHDI);

            // Apply equal weights to normalized values
                const weightedValue = (weightPop - 0.5*weightHDI) * normalizedPopulation + (weightHDI - 0.5*weightPop) * normalizedHDI;
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