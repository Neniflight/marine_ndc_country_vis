<script>
  // Write your JS here, or import other files
  import * as d3 from 'd3';
  import { onMount } from 'svelte';

  // Define variables
  let width, height, start;
  let maxClimate;
  let projection;

  let svg, globeGroup, path;

  let rotateEnabled = true; 

  $: console.log(maxClimate)

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
      .attr("width", width)
      .attr("height", height)

    globeGroup = svg.append("g");

    path = d3.geoPath().projection(projection);

    d3.json("countries-110m.json").then((world) => {
      globeGroup.append("path")
        .datum({type: "Sphere"})
        .attr("class", "ocean")
        .attr("d", path)
        .style('fill', '#132A50')

      
      d3.csv("topic_keyword_freq_by_countryndc_CN.csv").then(ndcData => {
      
      maxClimate = d3.max(ndcData, d => +d['climate change_freq_per_10000']);
      
      const colorScale = d3.scaleSequential()
        .domain([0, maxClimate]) // Input domain
        .interpolator(d3.interpolateBuGn);

      const ndcDataMap = new Map();
      ndcData.forEach(d => {
        const country = d.numeric_code
        const climate = +d['climate change_freq_per_10000'];

        ndcDataMap.set(country, climate)
        })

        function updateGlobeColors() {
          globeGroup.selectAll(".country")
          .transition().duration(1000)
          .style("fill", function (d) {
              let countryName = d.id;

              if (ndcDataMap.has(countryName)){
                let climate_freq;
                climate_freq = ndcDataMap.get(countryName);
                return colorScale(climate_freq);
              }
              else {
                  return "black";
              }
          })
        }

        globeGroup.selectAll(".country")
          .data(topojson.feature(world, world.objects.countries).features)
          .enter().append("path")
          .attr("class", "country")
          .attr("d", path)

        updateGlobeColors()
      });
    })

    svg.call(drag);
    svg.call(zoom);
    zoom.scaleTo(svg, 0.75)

    // d3.timer(rotateGlobe);
    
  });

</script>

<main>
  <h1>Svelte template</h1>

  <p>Write your HTML here</p>
  <div id="globe-container"></div>
  <script src="https://d3js.org/topojson.v3.min.js"></script>
</main>

<style>
  /* Write your CSS here */
</style>
