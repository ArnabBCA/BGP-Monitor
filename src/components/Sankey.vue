<template>
  <div class="graph">
    <svg ref="svg" class="svgContainer"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted, defineProps, watch, toRaw } from "vue";
import * as d3 from "d3";
import { sankeyCircular } from "d3-sankey-circular"; // Importing from d3-sankey-circular

const props = defineProps({
  nodesArray: Array,
  linksArray: Array,
});

const n = ref([]);
const l = ref([]);
const svgElement = ref(null);

watch(props, () => {
  n.value = JSON.parse(JSON.stringify(props.nodesArray));
  l.value = JSON.parse(JSON.stringify(props.linksArray));
});

watch([n, l], () => {
  svgElement.value.selectAll("*").remove();
  update();
});

const color = d3.scaleOrdinal(d3.schemeCategory10);
const linkColorScale = d3.scaleOrdinal(d3.schemeCategory10);

const getNodeColor = (d) => {
  return color(d.depth);
};

const update = () => {
  const { nodes, links } = sankeyCircular()
    .nodeId((d) => d.name)
    .nodeWidth(15)
    .nodePadding(15)
    .extent([
      [1, 5],
      [width - 1, height - 5],
    ])({
    nodes: n.value,
    links: l.value,
  });

  const rect = svgElement.value
    .append("g")
    .attr("stroke", "#000")
    .selectAll()
    .data(nodes)
    .join("rect")
    .attr("x", (d) => d.x0)
    .attr("y", (d) => d.y0)
    .attr("height", (d) => d.y1 - d.y0)
    .attr("width", (d) => d.x1 - d.x0)
    .attr("fill", getNodeColor);

  const link = svgElement.value
    .append("g")
    .attr("fill", "none")
    .attr("stroke-opacity", 0.5)
    .selectAll()
    .data(links)
    .join("g")
    .style("mix-blend-mode", "multiply");

  link
    .append("path")
    .attr("d", function (d) {
      return d.path;
    })
    .attr("stroke", (d) => linkColorScale(d.source.depth))
    .attr("stroke-width", (d) => Math.max(1, d.width));

  rect.append("title").text((d) => `${d.name}\n${format(d.value)} TWh`);

  svgElement.value
    .append("g")
    .selectAll()
    .data(nodes)
    .join("text")
    .attr("x", (d) => (d.x0 < width / 2 ? d.x1 + 6 : d.x0 - 6))
    .attr("y", (d) => (d.y1 + d.y0) / 2)
    .attr("dy", "0.35em")
    .attr("text-anchor", (d) => (d.x0 < width / 2 ? "start" : "end"))
    .text((d) => d.name);
};

const svg = ref(null);

const width = 1280;
const height = 720;
const format = d3.format(",.0f");

const init = () => {
  svgElement.value = d3
    .select(svg.value)
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .attr("style", "max-width: 100%; height: auto; font: 10px sans-serif;");
};

onMounted(() => {
  init();
});
</script>

<style scoped>
/*svg {
  padding-inline: max(2rem, calc(40% - 24rem));
}*/
</style>
