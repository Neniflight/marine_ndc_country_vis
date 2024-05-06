<script>
  // Write your JS here, or import other files
  import * as d3 from 'd3';
  import { onMount, afterUpdate } from 'svelte';

  // Define variables
  let width, height, start;
  let maxValue;
  let projection;
  let selectedTerm = "climate change";
  let world, barChartContainer;

  let svg, globeGroup, path;

  let rotateEnabled = true;

  onMount(() => {
    width = (2/3) * window.innerWidth;
    height = (2/3) * window.innerHeight;

    projection = d3.geoOrthographic()
      .scale(250)
      .translate([width / 2, height / 2])
      .clipAngle(90);
    const drag = d3.drag()
        .subject(function () {
            const r = projection.rotate();
            return { x: r[0] / 2, y: -r[1] / 2 };
        })
        .on("drag", function (event) {
          const rotate = projection.rotate();
          projection.rotate([rotate[0] + event.dx * 0.5, rotate[1] - event.dy * 0.5]);
          globeGroup.selectAll("path.country")
              .attr("d", path);
          rotateEnabled = false;
          start = Date.now();
        });

    const thresholdScale = 0.75;
    const initialTranslate = [width /2, height / 2];
    const initialZoomState = d3.zoomIdentity.translate(initialTranslate[0], initialTranslate[1]).scale(1);

    function calculateZoomTranslation(transform) {
        const scale = transform.k;
        const x = width / 2 - (width / 2) * scale;
        const y = height / 2 - (height / 2) * scale;
        return `translate(${x},${y}) scale(${scale})`;
    }

    const zoom = d3.zoom()
          .scaleExtent([0.75, 10]) // Set the minimum and maximum zoom levels
          .on("zoom", function (event) {
              globeGroup.attr("transform", calculateZoomTranslation(event.transform));
              // console.log("Zoom Scale:", event.transform.k);
    });

    function rotateGlobe() {
          if (rotateEnabled == false){return;}

          const rotate = projection.rotate();
          projection.rotate([rotate[0] + 0.1, rotate[1]]);
          globeGroup.selectAll("path.country")
              .attr("d", path);
    }

    svg  = d3.select("#globe-container")
      .append("svg")
      .attr("width", width )
      .attr("height", height)

    barChartContainer = d3.select("#bar-chart-container")
        .append("svg")
        .attr("width", width / 2 - 60) // Half the width for the bar chart
        .attr("height", height);

    globeGroup = svg.append("g");

    path = d3.geoPath().projection(projection);

    d3.json("countries-110m.json").then((geojsonData) => {
      world = geojsonData
      globeGroup.append("path")
        .datum({type: "Sphere"})
        .attr("class", "ocean")
        .attr("d", path)
        .style('fill', '#132A50')

        globeGroup.selectAll(".country")
          .data(topojson.feature(world, world.objects.countries).features)
          .enter().append("path")
          .attr("class", "country")
          .attr("d", path)

        updateVisualization();
    })

    svg.call(drag);
    svg.call(zoom);
    zoom.scaleTo(svg, 0.75)

    d3.timer(rotateGlobe);
  });

  afterUpdate(() => {
    updateVisualization();
  });

  function updateVisualization() {
    d3.csv("topic_keyword_freq_by_countryndc_CN.csv").then(ndcData => {
        maxValue = d3.max(ndcData, d => +d[selectedTerm + "_freq_per_10000"]);
        console.log(maxValue);
        const colorScale = d3.scaleSequential()
          .domain([0, maxValue])
          .interpolator(d3.interpolateBuGn);

        const ndcDataMap = new Map();
        ndcData.forEach(d => {
          const country = d.numeric_code;
          const value = +d[selectedTerm + "_freq_per_10000"];
          ndcDataMap.set(country, value);
        });

        function updateGlobeColors() {
          globeGroup.selectAll(".country")
            .data(topojson.feature(world, world.objects.countries).features)
            .join("path")
            .attr("class", "country")
            .attr("d", path)
            // .transition() for some reason, the bar chart does not appear when i uncomment this
            // .duration(1000)
            .style("fill", function (d) {
              let countryID = d.id;
              if (ndcDataMap.has(countryID)) {
                let climateFreq = ndcDataMap.get(countryID);
                return colorScale(climateFreq);
              } else {
                return "black";
              }
            })
            .on("mouseover", function(event, d) {
              let countryID = d.id
              let countryName = d.properties.name;
              let termFreq = ndcDataMap.get(countryID).toFixed(2) || 0;
              d3.select(".tooltip")
                .style("display", "block")
                .html(`<strong>${countryName}</strong><br>Term Frequency: ${termFreq}`);
            })
            .on("mousemove", function(event) {
              d3.select(".tooltip")
                .style("left", (event.pageX + 10) + "px")
                .style("top", (event.pageY - 10) + "px");
            })
            .on("mouseout", function() {
              d3.select(".tooltip").style("display", "none");
            });
        }
        updateGlobeColors();

        barChartContainer.selectAll("*").remove();

        ndcData.sort((a, b) => b[selectedTerm + "_freq_per_10000"] - a[selectedTerm + "_freq_per_10000"]);
        const topCountries = ndcData.slice(0, 10);

        const xScale = d3.scaleBand()
          .domain(topCountries.map(d => d.country))
          .range([-6, width * 0.5 + 60])
          .padding(0.1);

        const yScale = d3.scaleLinear()
          .domain([0, maxValue])
          .range([height * 0.5, 0]);

        const xAxis = d3.axisBottom(xScale)
          .tickSize(0)
        
        const xAxisGroup = barChartContainer.append("g")
          .attr("transform", "translate(0," + (height * 0.5) +  ")")
          .call(xAxis)
          .selectAll("text")
          .attr("transform", "translate(20, 0) rotate(-60)")
          .style("text-anchor", "end");

        const yAxis = d3.axisLeft(yScale)
          .ticks(5)
          .tickSizeInner(-width * 0.5);

        const yAxisGroup = barChartContainer.append("g")
          .call(yAxis)
          .attr("transform", "translate(20, 0)");

        barChartContainer.selectAll(".bar")
          .data(ndcData)
          .enter().append("rect")
          .attr("class", "bar")
          .attr("x", d => xScale(d.country))
          .attr("width", xScale.bandwidth())
          .attr("y", d => yScale(+d[selectedTerm + "_freq_per_10000"]))
          .attr("height", d => height * 0.5 - yScale(+d[selectedTerm + "_freq_per_10000"]))
          .attr("fill", "steelblue")
          .attr("transform", "translate(20, 0)");
      });


}
</script>

<main>
  <h1>Globe Visualization for NDCs word frequencies!</h1>
  <div class="container">
    <div id="globe-container">
      <div class="tooltip"></div>
    </div>
    <div id="bar-chart-container"></div>
  </div>
  <script src="https://d3js.org/topojson.v3.min.js"></script>
  <select id="term" bind:value={selectedTerm}>
    <option value="climate change">climate change</option>
    <option value="coastal impacts">coastal impacts</option>
    <option value="ocean ecosystems">ocean ecosystems</option>
    <option value="biodiversity">biodiversity</option>
    <option value="pollutants/waste">pollutants/waste</option>
    <option value="fisheries">fisheries</option>
    <option value="industry">industry</option>
    <option value="blue economy">blue economy</option>
    <option value="policy">policy</option>
  </select>
</main>

<style>
  .tooltip {
    position: absolute;
    background-color: rgba(255, 255, 255, 0.8);
    padding: 5px;
    border-radius: 5px;
    pointer-events: none;
    display: none;
  }

  .container {
    display: flex;

  }
</style>
