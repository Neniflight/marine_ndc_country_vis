<script>
	import * as d3 from 'd3';
	import { onMount, afterUpdate } from 'svelte';

	export let selectedTerm;
	let width, height, projection, svg, globeGroup, path, world, legendSvg, maxValue;

	let rotateEnabled = true;

	onMount(() => {
		width = (0.5) * window.innerWidth;
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
				const zoomScale = d3.zoomTransform(svg.node()).k; 
				const sensitivityFactor = 0.5 / zoomScale; 
				const rotate = projection.rotate();
				projection.rotate([rotate[0] + event.dx * sensitivityFactor, rotate[1] - event.dy * sensitivityFactor]);
				globeGroup.selectAll("path.country")
					.attr("d", path);
				rotateEnabled = false;
			});

		const initialTranslate = [width /2, height / 2];

		function calculateZoomTranslation(transform) {
			const scale = transform.k;
			const x = width / 2 - (width / 2) * scale;
			const y = height / 2 - (height / 2) * scale;
			return `translate(${x},${y}) scale(${scale})`;
    }

		const zoom = d3.zoom()
			.scaleExtent([0.25, 10]) // Set the minimum and maximum zoom levels
			.on("zoom", function (event) {
				globeGroup.attr("transform", calculateZoomTranslation(event.transform));
    });

		function rotateGlobe () {
			if (rotateEnabled == false) {return;}

			const rotate = projection.rotate();
			projection.rotate([rotate[0] + 0.1, rotate[1]]);
			globeGroup.selectAll("path.country")
				.attr("d", path);
		}

		svg  = d3.select("#globe-container")
      .append("svg")
      .attr("width", width )
      .attr("height", height)

		globeGroup = svg.append("g");
		path = d3.geoPath().projection(projection);

		legendSvg = d3.select("#legend");

		d3.json("countries-110m.json").then((geojsonData) => {
      world = geojsonData
      globeGroup.append("path")
        .datum({type: "Sphere"})
        .attr("class", "ocean")
        .attr("d", path)
        .style('fill', '#D3D3D3')
        .style("stroke", "black")
        .style("stroke-width", 1);

      globeGroup.selectAll(".country")
        .data(topojson.feature(world, world.objects.countries).features)
        .enter().append("path")
        .attr("class", "country")
        .attr("d", path)
        .style("stroke", "black")
        .style("stroke-width", 0.5);

      updateVisualization();
    })
			
		svg.call(drag);
    svg.call(zoom);
    zoom.scaleTo(svg, 0.75)
    d3.timer(rotateGlobe);
	})

	function updateVisualization() {
    d3.csv("topic_keyword_freq_by_countryndc_CN.csv").then(ndcData => {
      maxValue = d3.max(ndcData, d => +d[selectedTerm + "_freq_per_10000"]/10000*d['num_words']);
      const colorScale = d3.scaleSequential()
        .domain([0, maxValue])
        .interpolator(d3.interpolateYlGnBu);

      const ndcDataMap = new Map();
      ndcData.forEach(d => {
        const country = d.numeric_code;
        const value = +d[selectedTerm + "_freq_per_10000"]/10000*d['num_words'];
        ndcDataMap.set(country, value);
      });

      globeGroup.selectAll(".country")
        .data(topojson.feature(world, world.objects.countries).features)
        .join("path")
        .attr("class", "country")
        .attr("d", path)
        .transition()
        .duration(1000)
        .style("fill", d => {
          const countryID = d.id;
					if (ndcDataMap.has(countryID)) {
               let climateFreq = ndcDataMap.get(countryID);
               return colorScale(climateFreq);
             } else {
               return "gray";
             }
        });

      globeGroup.selectAll(".country")
        .on("mouseover", (event, d) => {
          const countryID = d.id;
          const countryName = d.properties.name;
          let termFreq = ndcDataMap.get(countryID) || 0;
					console.log(termFreq);
					if (termFreq !== undefined) {
						termFreq = Math.round((termFreq + Number.EPSILON) * 100) / 100
					} else {
							termFreq = 0;
					}
          d3.select(".tooltip")
            .html(`<strong>${countryName}</strong><br>Term Count: ${termFreq.toFixed(2)}`)
            .style("display", "block");
        })
        .on("mousemove", event => {
          d3.select(".tooltip")
            .style("left", (event.pageX + 10) + "px")
            .style("top", (event.pageY - 10) + "px");
        })
        .on("mouseout", () => {
          d3.select(".tooltip").style("display", "none");
        });
				updateLegend();
    });
	}

	function updateLegend() {
		legendSvg.selectAll("*").remove();
    const legendWidth = 380;
    const legendHeight = 20;
    const legendPadding = 20;

    const colorScale = d3.scaleSequential()
      .domain([0, maxValue])
      .interpolator(d3.interpolateYlGnBu);

    const numStops = 10;

    const gradientData = d3.range(0, numStops).map(d => d / (numStops - 1));

    const legendGradient = legendSvg.append("defs")
      .append("linearGradient")
      .attr("id", "legendGradient")
      .attr("x1", "0%")
      .attr("y1", "0%")
      .attr("x2", "100%")
      .attr("y2", "0%");

    legendGradient.selectAll("stop")
      .data(gradientData)
      .enter().append("stop")
      .attr("offset", d => d * 100 + "%")
      .attr("stop-color", d => colorScale(d * maxValue));

    legendSvg.append("rect")
      .attr("x", legendPadding)
      .attr("y", legendPadding)
      .attr("width", legendWidth)
      .attr("height", legendHeight)
      .style("fill", "url(#legendGradient)");

    const legendScale = d3.scaleLinear()
      .domain([0, maxValue])
      .range([0, legendWidth]);

    const legendAxis = d3.axisBottom(legendScale)
      .ticks(5)
      .tickSize(5)
      .tickFormat(d3.format(".0f"));

    legendSvg.append("g")
      .attr("class", "legend-axis")
      .attr("transform", `translate(${legendPadding}, ${legendHeight + legendPadding + 5})`)
      .call(legendAxis);

    legendSvg.append("text")
      .attr("x", legendPadding)
      .attr("y", legendPadding - 5)
      .text("Count");
  }

    afterUpdate(() => {
        console.log(selectedTerm);
        updateVisualization();
    }) 

</script>
<div class="vert-container">
	<div id="globe-container">
    <script src="https://d3js.org/topojson.v3.min.js"></script>
	<div class="tooltip"></div>
	</div>
	<svg id="legend" width="50%"></svg>
</div>


<style>
	.tooltip {
    position: absolute;
    background-color: rgba(255, 255, 255, 0.8);
    padding: 5px;
    border-radius: 5px;
    pointer-events: none;
    display: none;
  }

	.vert-container {
		display: flex;
		flex-direction: column;
		align-items: center;
	}
</style>