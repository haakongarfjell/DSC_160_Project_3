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
    let weight = 0.5;
    let populationData; 

    onMount(async () => {
      svg = d3.select("#my_dataviz");
      width = window.innerWidth;
      height = window.innerHeight * 0.9;
  
      projection = d3.geoMercator()
        .scale(150)
        .translate([width / 2, height / 1.5]);
  
      path = d3.geoPath().projection(projection);
  
      colorScale = d3.scaleThreshold()
        .domain([100000, 1000000, 10000000, 30000000, 100000000, 500000000])
        .range(d3.schemeBlues[7]);
  
      const tooltip = d3.select("body").append("div")
        .attr("class", "tooltip")
        .style("opacity", 0);
  
      const [topoData, popData] = await Promise.all([
          d3.json("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson"),
          d3.csv("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world_population.csv")
      ]);

      populationData = new Map(popData.map(d => [d.code, +d.pop]));

      svg.attr("width", width)
         .attr("height", height);
  
      updateMap(topoData);

      d3.select("#weightSlider").on("input", function() {
        weight = +this.value;
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
            .style("fill", function(d) {
                const population = populationData.get(d.id) || 0;
                const secondPopulation = secondData.get(d.id) || 0;
                const weightedPopulation = weight * population + (1 - weight) * secondPopulation;
                return colorScale(weightedPopulation);
            })
            .on("mouseover", function(d) {
                const population = populationData.get(d.id) || 0;
                d3.select(this).style("fill", "#ffab00");
                tooltip.transition()
                    .duration(200)
                    .style("opacity", .9);
                tooltip.html("Country: " + d.properties.name + "<br>Population: " + population.toLocaleString())
                    .style("left", (d3.event.pageX + 10) + "px")
                    .style("top", (d3.event.pageY - 28) + "px");
            })
            .on("mouseout", function(d) {
                d3.select(this).style("fill", function(d) {
                    const population = populationData.get(d.id) || 0;
                    const secondPopulation = secondData.get(d.id) || 0;
                    const weightedPopulation = weight * population + (1 - weight) * secondPopulation;
                    return colorScale(weightedPopulation);
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
      left: 20px;
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
    }
  </style>
  
  <svg id="my_dataviz"></svg>
  
  <div class="slider-container">
    <input type="range" min="0" max="1" value="0.5" step="0.01" class="slider" id="weightSlider">
  </div>
  