<script>
  import Globe from './Globe.svelte';
  import BarChart from './BarChart.svelte';

  let selectedTerm = "climate change";
  let selectedCount = "word counts"
</script>

<main>
  <h1>Globe Visualization for NDCs word frequencies!</h1>
  <div class="container">
    <Globe {selectedTerm} />
    <BarChart {selectedTerm} />
  </div>
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
  <select id="frequency_count" bind:value={selectedCount}>
    <option value="frequency per 10k words">frequency per 10k words</option>
    <option value="word counts">word counts</option>
  </select>
</main>

<style>
  .container {
    display: flex;
  }

  main {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  
</style>