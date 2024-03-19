<template>
  <div class="graph">
    <div class="svgContainer">
      <svg ref="svgRef" width="720" height="480"></svg>
    </div>
  </div>
</template>

<script setup>
import * as d3 from "d3";
import dagre from "dagre/dist/dagre.min.js"
import { ref, onMounted, watch, toRaw } from "vue";

const svgRef = ref(null);
const simulation = ref(null);
const g_links = ref(null);
const g_nodes = ref(null);
const g_label = ref(null);

const props = defineProps({
  nodesArray: Array,
  linksArray: Array,
  oiginNodeId: Number,
});

const init = () => {
  const svg = d3.select(svgRef.value);
  svg.call(d3.zoom().on("zoom", zoomed));
  const width = +svg.attr("width");
  const height = +svg.attr("height");

  positionWithDagre(); // Position nodes using Dagre.js

  simulation.value = d3
    .forceSimulation(toRaw(props.nodesArray))
    .force("charge", d3.forceManyBody().strength(-300))
    .force(
      "link",
      d3
        .forceLink()
        .id(function (d) {
          return d.id;
        })
        .links(toRaw(props.linksArray))
    )
    .force("center", d3.forceCenter(width / 2, height / 2))
    .on("tick", ticked);

  g_links.value = svg.append("g").attr("class", "links");
  g_nodes.value = svg.append("g").attr("class", "nodes");
  g_label.value = svg.append("g").attr("class", "labels");

  reDraw(); // Draw the graph with updated positions
};

const positionWithDagre = () => {
  const g = new dagre.graphlib.Graph();
  g.setGraph({ rankdir: 'BT' }); // 'TB' for top to bottom layout
  g.setDefaultEdgeLabel(() => ({}));

  // Add nodes to the graph
  props.nodesArray.forEach(node => {
    g.setNode(node.id, { width: 50, height: 50 }); // You might want to adjust width and height
  });

  // Add edges to the graph
  props.linksArray.forEach(link => {
    g.setEdge(link.source, link.target);
  });

  // Run the layout
  dagre.layout(g);

  // Update node positions
  props.nodesArray.forEach(node => {
    const layoutNode = g.node(node.id);
    node.x = layoutNode.x;
    node.y = layoutNode.y;
  });
};

const ticked = () => {
  g_label.value
    .selectAll("text")
    .attr("x", function (d) {
      return d.x;
    })
    .attr("y", function (d) {
      return d.y;
    });
  g_nodes.value
    .selectAll("circle")
    .attr("cx", (d) => {
      return d.x;
    })
    .attr("cy", (d) => {
      return d.y;
    });

  g_links.value
    .selectAll("line")
    .attr("x2", (d) => {
      return d.source.x;
    })
    .attr("y2", (d) => {
      return d.source.y;
    })
    .attr("x1", (d) => {
      return d.target.x;
    })
    .attr("y1", (d) => {
      return d.target.y;
    });
};

function dragstarted(event) {
  simulation.value.alphaTarget(0.3).restart();
  event.subject.fx = event.subject.x;
  event.subject.fy = event.subject.y;
}

function dragged(event) {
  event.subject.fx = event.x;
  event.subject.fy = event.y;
}

function dragended(event) {
  simulation.value.alphaTarget(0);
  event.subject.fx = null;
  event.subject.fy = null;
}

function zoomed(event) {
  g_nodes.value.attr("transform", event.transform);
  g_links.value.attr("transform", event.transform);
  g_label.value.attr("transform", event.transform);
}

const reDraw = () => {
  const update_nodes = g_nodes.value.selectAll("circle").data(toRaw(props.nodesArray));
  update_nodes.exit().remove();
  update_nodes
    .enter()
    .append("circle")
    .merge(update_nodes)
    .style("fill", (d) =>{
      if(d.id === props.oiginNodeId){
        return "#FDB750"
      }
    })
    .call(
      d3
        .drag()
        .on("start", dragstarted)
        .on("drag", dragged)
        .on("end", dragended)
    );

  const update_links = g_links.value.selectAll("line").data(toRaw(props.linksArray));
  update_links.exit().remove();
  update_links.enter().append("line").merge(update_links);

  const update_labels = g_label.value.selectAll("text").data(toRaw(props.nodesArray));
  update_labels.exit().remove();
  update_labels
    .enter()
    .append("text")
    .merge(update_labels)
    .text(function (d) {
      return d.name;
    });
};

const update = () => {
  positionWithDagre();
  simulation.value.nodes(toRaw(props.nodesArray));
  simulation.value.force(
    "link",
    d3
      .forceLink()
      .id(function (d) {
        return d.id;
      })
      .links(toRaw(props.linksArray))
  );
  simulation.value.alpha(1).restart();
  reDraw();
};

onMounted(() => {
  init()
})
watch(props, () => {
  update();
});
</script>

<style>
  .graph {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
  }

  .svgContainer {
    border: 1px solid #ccc;
  }
</style>
