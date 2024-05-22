<script>
	import * as d3 from 'd3';
	import { onMount, afterUpdate} from 'svelte';

	export let selectedTerm;
  let width, height, barChartContainer;

	onMount(() => {
    width = (0.4) * window.innerWidth;
    height = (2 / 3) * window.innerHeight;

    barChartContainer = d3.select("#bar-chart-container")
      .append("svg")
      .attr("width", width + 40)
      .attr("height", height);

  });

	function updateBarChart() {
		barChartContainer.selectAll("*").remove();

		d3.csv("topic_keyword_freq_by_countryndc_CN.csv").then(ndcData => {
			const maxValue = d3.max(ndcData, d => +d[selectedTerm + "_freq_per_10000"]/10000*d['num_words']);
      ndcData.sort((a, b) => (b[selectedTerm + "_freq_per_10000"]/10000*b['num_words'] - a[selectedTerm + "_freq_per_10000"]/10000*a['num_words']));
      const topCountries = ndcData.slice(0, 10);

			const xScale = d3.scaleBand()
				.domain(topCountries.map(d => d.country))
				.range([0, width])
				.padding(0.3)

			const yScale = d3.scaleLinear()
				.domain([0, maxValue+5])
				.range([height * 0.5, 0]);

			const xAxis = d3.axisBottom(xScale).tickSize(0)
			const yAxis = d3.axisLeft(yScale).tickSize(5).tickSizeInner(-width);

			barChartContainer.append("g")
        .attr("transform", "translate(40," + (height * 0.5) + ")")
        .call(xAxis)
        .selectAll("text")
        .attr("transform", "translate(0, 10) rotate(-60)")
        .attr("font-size", "0.8rem")
        .style("text-anchor", "end");

			barChartContainer.append("g")
        .call(yAxis)
        .attr("transform", "translate(40, 0)");

			barChartContainer.append("text")
        .attr("class", "x label")
        .attr("text-anchor", "end")
        .attr("x", width * 0.25 + 100)
        .attr("y", height * 0.65)
        .attr("transform", "translate(40, 40)")
        .text("Country");

			barChartContainer.append("text")
        .attr("class", "y label")
        .attr("text-anchor", "middle")
        .attr("x", -height * 0.25-10)
        .attr("y", 15)
        .attr("transform", "rotate(-90)")
        .text("Word Count by Country");

			barChartContainer.selectAll(".bar")
				.data(topCountries)
				.enter().append("rect")
				.attr("class", "bar")
				.attr("x", d => xScale(d.country))
				.attr("width", xScale.bandwidth())
				.attr("y", height * 0.5)
				.attr("height", 0)
				.attr("fill", "steelblue")
				.attr("transform", "translate(40, 0)")
				.transition()
				.duration(1000)
				.attr("y", d => yScale(+d[selectedTerm + "_freq_per_10000"]/10000*d['num_words']))
				.attr("height", d => height * 0.5 - yScale(+d[selectedTerm + "_freq_per_10000"]/10000*d['num_words']));
		})
	}

	afterUpdate(() => {
        console.log(selectedTerm);
        updateBarChart();
    }) 

</script>

<div id="bar-chart-container"></div>