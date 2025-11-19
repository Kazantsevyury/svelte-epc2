<script>
  import * as Plot from "@observablehq/plot";
  import { onMount } from "svelte";
  import rawData from "./ppd_complaints.json";

  // =============================
  // 1. Normalize Race
  // =============================
  function normalizeRace(r) {
    if (!r) return "Other";
    const R = r.trim().toLowerCase();
    if (R === "white") return "White";
    if (R === "black") return "Black";
    if (R === "hispanic" || R === "latino") return "Hispanic";
    return "Other";
  }

  // =============================
  // 2. Aggregate Data
  // =============================
  const raceOrder = ["White", "Black", "Hispanic", "Other"];

  function buildRaceStats(data) {
    const map = new Map();

    for (const d of data) {
      const race = normalizeRace(d.po_race);
      const id = d.officer_id;
      const finding = d.investigative_findings;

      if (!map.has(race)) {
        map.set(race, {
          race,
          complaints: 0,
          officers: new Set(),
          sustained: 0
        });
      }

      const e = map.get(race);
      e.complaints += 1;
      if (id != null) e.officers.add(id);
      if (finding === "Sustained Finding") e.sustained += 1;
    }

    const stats = [];
    for (const e of map.values()) {
      const officers = e.officers.size || 1;
      const cpo = e.complaints / officers;
      const spo = e.complaints ? e.sustained / e.complaints : 0;

      stats.push({
        race: e.race,
        complaints: e.complaints,
        officers,
        cpo,
        spo
      });
    }

    stats.sort((a, b) => raceOrder.indexOf(a.race) - raceOrder.indexOf(b.race));
    return stats;
  }

  const raceStats = buildRaceStats(rawData);

  // =============================
  // 3. Colors
  // =============================
  const colorByRace = {
    White: "#d62728",
    Black: "#1f77b4",
    Hispanic: "#ffbf00",
    Other: "#7f7f7f"
  };

  // =============================
  // 4. DOM Refs
  // =============================
  let bubbleChartEl;
  let cpoChartEl;
  let spoChartEl;

  // =============================
  // 5. Render Charts
  // =============================
  function renderBubbleChart() {
    if (!bubbleChartEl) return;
    bubbleChartEl.innerHTML = "";

    const chart = Plot.plot({
      width: 960,
      height: 360,
      marginLeft: 80,
      marginBottom: 60,
      style: { background: "white" },
      marks: [
        Plot.dot(raceStats, {
          x: "cpo",
          y: "spo",
          r: d => d.complaints * 2,
          fill: d => colorByRace[d.race],
          stroke: "black",
          strokeWidth: 1,
          title: d =>
            `${d.race}
Complaints: ${d.complaints}
Officers: ${d.officers}
Complaints/Officer: ${d.cpo.toFixed(2)}
Sustained Share: ${(d.spo * 100).toFixed(1)}%`
        }),

        Plot.text(raceStats, {
          x: "cpo",
          y: "spo",
          text: "race",
          dy: -10,
          fontSize: 13,
          fill: "black"
        })
      ],
      x: { label: "Complaints per Officer", grid: true },
      y: {
        label: "Sustained Share (Sustained / Complaints)",
        tickFormat: d => (d * 100).toFixed(0) + "%",
        grid: true
      }
    });

    bubbleChartEl.append(chart);
  }

  function renderCpoChart() {
    if (!cpoChartEl) return;
    cpoChartEl.innerHTML = "";

    const chart = Plot.plot({
      width: 440,
      height: 280,
      marginBottom: 60,
      style: { background: "white" },
      marks: [
        Plot.barY(raceStats, {
          x: "race",
          y: "cpo",
          fill: d => colorByRace[d.race],
          title: d =>
            `${d.race}
Complaints per Officer: ${d.cpo.toFixed(2)}
Total Complaints: ${d.complaints}`
        }),

        Plot.ruleY([0])
      ],
      x: { label: "Officer Race" },
      y: { label: "Complaints per Officer", grid: true }
    });

    cpoChartEl.append(chart);
  }

  function renderSpoChart() {
    if (!spoChartEl) return;
    spoChartEl.innerHTML = "";

    const chart = Plot.plot({
      width: 440,
      height: 280,
      marginBottom: 60,
      style: { background: "white" },
      marks: [
        Plot.barY(raceStats, {
          x: "race",
          y: "spo",
          fill: d => colorByRace[d.race],
          title: d =>
            `${d.race}
Sustained Share: ${(d.spo * 100).toFixed(1)}%
Total Sustained: ${(d.complaints * d.spo).toFixed(0)}`
        }),

        Plot.ruleY([0])
      ],
      x: { label: "Officer Race" },
      y: {
        label: "Sustained Share",
        tickFormat: d => (d * 100).toFixed(0) + "%",
        grid: true
      }
    });

    spoChartEl.append(chart);
  }

  onMount(() => {
    renderBubbleChart();
    renderCpoChart();
    renderSpoChart();
  });
</script>
<style>
  :root {
    font-family: system-ui, sans-serif;
    background: #222;
    color: #f4f4f4;
    line-height: 1.5;
  }

  body {
    margin: 0;
  }

  .dashboard {
    width: 90%;
    max-width: 1200px;
    margin: 0 auto;
    padding: 24px 0 48px;
  }

  .header {
    text-align: center;
    margin-bottom: 24px;
  }

  .header h1 {
    font-size: 2.2rem;
    margin: 0 0 4px;
  }

  .header h2 {
    font-size: 1.05rem;
    opacity: 0.85;
    margin: 0 0 4px;
  }

  .header p {
    font-size: 0.9rem;
    opacity: 0.7;
    margin: 0;
  }

  .row {
    display: grid;
    gap: 24px;
    margin-bottom: 24px;
  }

  .row-bottom {
    grid-template-columns: 1fr 1fr;
  }

  .panel {
    background: white;
    color: black;
    padding: 16px;
    border-radius: 10px;
  }

  .panel h3 {
    margin-top: 0;
    margin-bottom: 8px;
    font-size: 1.05rem;
  }

  .chart-container {
    overflow: hidden;
  }

  /* ===== GLOBAL HOVER EFFECT (works because of :global selector) ===== */

  :global(.chart-container svg *:hover) {
    filter: brightness(1.4);
    opacity: 1 !important;
    cursor: pointer;
    transition: filter 0.15s ease-out, opacity 0.15s ease-out;
  }

  :global(.chart-container svg *) {
    transition: filter 0.15s ease-out, opacity 0.15s ease-out;
  }
</style>



<div class="dashboard">
  <header class="header">
    <h1>Police Complaints — 2021</h1>
    <h2>Complaints and Sustained Findings per Officer Race</h2>
    <p>
      Hover any bar or bubble to see details. Tooltips are generated by Plot automatically.
    </p>
  </header>

  <div class="row">
    <section class="panel">
      <h3>Complaints vs Sustained per Officer (by Race)</h3>
      <div class="chart-container" bind:this={bubbleChartEl}></div>
    </section>
  </div>

  <div class="row row-bottom">
    <section class="panel">
      <h3>Complaints per Officer by Race</h3>
      <div class="chart-container" bind:this={cpoChartEl}></div>
    </section>
    <section class="panel">
      <h3>Sustained Share by Race</h3>
      <div class="chart-container" bind:this={spoChartEl}></div>
    </section>
  </div>
</div>
