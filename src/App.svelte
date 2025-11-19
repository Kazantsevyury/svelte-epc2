<script>
  import * as Plot from "@observablehq/plot";
  import { onMount } from "svelte";
  import rawData from "./ppd_complaints.json";
  
  const colorByRace = {
    White: "#d62728",
    Black: "#1f77b4",
    Latino: "#ffbf00",
    Other: "#7f7f7f"
  };

  // =============================
  // Aggregate race data
  // =============================
  const raceOrder = ["White", "Black", "Latino", "Other"];

  function normalizeRace(r) {
    if (!r) return "Other";
    const R = r.trim().toLowerCase();
    if (R === "white") return "White";
    if (R === "black") return "Black";
    if (R === "latino") return "Latino";
    return "Other";
  }

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
  // Legend
  // =============================
  let legendEl;
  
  function renderLegend() {
    legendEl.innerHTML = `
      <div style="display:flex; gap:20px; margin:6px 0 10px;">
        ${raceOrder
          .map(
            r => `
          <div style="display:flex; align-items:center; gap:6px;">
            <div style="width:14px; height:14px; background:${colorByRace[r]};
                 border-radius:3px;"></div>
            <span style="font-size:0.9rem;">${r}</span>
          </div>`
          )
          .join("")}
      </div>`;
  }


  let bubbleChartEl;
  let cpoChartEl;
  let spoChartEl;

  function renderBubbleChart() {
  bubbleChartEl.innerHTML = "";

  const maxCpo = Math.max(...raceStats.map(d => d.cpo));
  const minCpo = Math.min(...raceStats.map(d => d.cpo));
  const maxSpo = Math.max(...raceStats.map(d => d.spo));

  const chart = Plot.plot({
    width: 960,
    height: 380,
    marginLeft: 60,
    marginRight: 60,
    marginTop: 30,
    marginBottom: 80,
    style: { background: "white" },
    marks: [

      Plot.text(
        [{
          x: (maxCpo + minCpo) / 2,
          y: maxSpo * 0.98,
          label: "Bubble size = number of complaints"
        }],
        {
          x: "x",
          y: "y",
          text: "label",
          textAnchor: "middle",
          fontSize: 12,
          fill: "#444"
        }
      ),

      Plot.dot(raceStats, {
        x: "cpo",
        y: "spo",
        r: d => Math.sqrt(d.complaints) * 6,
        fill: d => colorByRace[d.race],
        stroke: "black",
        strokeWidth: 1.1,
        opacity: 0.95,
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
        dy: -18,
        fontSize: 13,
        fill: "black",
        stroke: "white",
        strokeWidth: 2.2
      })
    ],

    x: { 
      label: "Complaints per Officer", 
      grid: true,
      domain: [1.7, 2.4]    
    },

    y: {
      label: "Sustained Share (Sustained / Complaints)",
      tickFormat: d => (d * 100).toFixed(0) + "%",
      grid: true
    }
  });

  bubbleChartEl.append(chart);
}



  function renderCpoChart() {
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
    renderLegend();
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
    margin-bottom: 4px;
    text-align: center; 
  }

  .chart-container {
    overflow: hidden;
  }

  :global(.chart-container svg *:hover) {
    filter: brightness(1.4);
    cursor: pointer;
    transition: filter 0.15s ease-out;
  }
</style>

<div class="dashboard">
  <header class="header">
    <h1>Police Complaints — 2021</h1>
    <h2>Complaints and Sustained Findings per Officer Race</h2>
    <p>Hover any bar or bubble to see details.</p>
  </header>

  <div class="row">
    <section class="panel">
      <h3>Complaints vs Sustained per Officer (by Race)</h3>
      <div class="legend" bind:this={legendEl}></div>
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