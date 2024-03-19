<script setup>
import { ref, onMounted } from "vue";
import Plotly from "plotly.js-dist";

const plot = ref(null);
const chartData = ref({
  x: [],
  y: [],
  type: "line",
  line: { shape: "linear" },
  marker: { size: 8 },
});
const chartLayout = ref({
  title: "Messages per second",
  xaxis: { title: "Timestamp" },
  yaxis: { title: "Ratio", rangemode: "tozero" },
});

function getData() {
  return Math.random();
}

var cnt = 0;

setInterval(function () {
  Plotly.extendTraces(plot.value, { y: [[getData()]] }, [0]);
  cnt++;
  /*Plotly.relayout(plot.value, {
    xaxis: {
      range: [cnt - 500, cnt],
    },
  });*/
}, 0);

onMounted(() => {
  Plotly.newPlot(plot.value, [
    {
      y: [getData()],
      type: "line",
    },
  ]);
});
</script>

<template>
  <div>
    <div ref="plot"></div>
  </div>
</template>
