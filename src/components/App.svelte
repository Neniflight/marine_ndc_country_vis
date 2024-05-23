<script>
  import Globe from './Globe.svelte';
  import BarChart from './BarChart.svelte';

  let selectedTerm = "climate change";
  let selectedCount = true;
</script>

<main class="background">
  <h1>Globe Visualization for NDCs word frequencies!</h1>
  <div class="container">
    <Globe {selectedTerm} {selectedCount}/>
    <BarChart {selectedTerm} {selectedCount}/>
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
  <!-- <select id="frequency_count" bind:value={selectedCount}>
    <option value={true}>word counts</option>
    <option value={false}>frequency per 10k words</option>
  </select> -->
  <label>
    <input type="checkbox" bind:checked={selectedCount} />
    <!-- {selectedCount ? 'Word counts' : 'Frequency per 10k words'} -->
    Word Counts
  </label>
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