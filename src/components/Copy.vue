<script setup>
import * as d3 from "d3";
import { ref, onMounted, watch, toRaw } from "vue";

const svgRef = ref(null);
const simulation = ref(null);
const g_links = ref(null);
const g_nodes = ref(null);
const g_label = ref(null);

const newNodes = ref([]);
const deleteNodes = ref([]);
const newLinks = ref([]);
const deleteLinks = ref([]);

const props = defineProps({
  nodesArray: Array,
  linksArray: Array,
  oiginNodeId: Number,
});

watch(
  () => props.nodesArray,
  (newNodesArray, oldNodesArray) => {
    // Clear the arrays
    newNodes.value = [];
    deleteNodes.value = [];

    // Check for new and deleted nodes
    newNodesArray.forEach(node => {
      // Check if the node is not present in the oldNodesArray
      if (!oldNodesArray.some(oldNode => oldNode.id === node.id)) {
        // Add the node to the newNodes array
        newNodes.value.push(node);
      }
    });
    oldNodesArray.forEach(oldNode => {
      // Check if the node is not present in the newNodesArray
      if (!newNodesArray.some(node => node.id === oldNode.id)) {
        // Add the node to the deleteNodes array
        deleteNodes.value.push(oldNode);
      }
    });
    console.log("newNodes", newNodes.value.map(node => node.id));
    console.log("deleteNodes", deleteNodes.value.map(node => node.id));
  },
  { deep: true }
);

const init = () => {
  const svg = d3.select(svgRef.value);
  svg.call(d3.zoom().on("zoom", zoomed));
  const width = +svg.attr("width");
  const height = +svg.attr("height");

  simulation.value = d3
    .forceSimulation()
    .force("charge", d3.forceManyBody().strength(-300))
    .force("center", d3.forceCenter(width / 2, height / 2))
    .on("tick", ticked);

  g_links.value = svg.append("g").attr("class", "links");
  g_nodes.value = svg.append("g").attr("class", "nodes");
  g_label.value = svg.append("g").attr("class", "labels");
};

const ticked = () => {
  //const alphaValue = simulation.value.alpha();
  //const k = 6 * alphaValue;
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
    //.each(function(d) { d.source.y -= k, d.target.y += k; })
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
  const update_nodes = g_nodes.value
    .selectAll("circle")
    .data(toRaw(props.nodesArray));
  update_nodes.exit().remove();
  update_nodes
    .enter()
    .append("circle")
    .merge(update_nodes)
    .style("fill", (d) => {
      if (d.id === props.oiginNodeId) {
        return "#FDB750";
      }
    })
    .call(
      d3
        .drag()
        .on("start", dragstarted)
        .on("drag", dragged)
        .on("end", dragended)
    );

  const update_links = g_links.value
    .selectAll("line")
    .data(toRaw(props.linksArray));
  update_links.exit().remove();
  update_links.enter().append("line").merge(update_links);

  const update_labels = g_label.value
    .selectAll("text")
    .data(toRaw(props.nodesArray));
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
  simulation.value.nodes(toRaw(props.nodesArray));
  simulation.value.force(
    "link",
    d3
      .forceLink()
      .id(function (d) {
        return d.id;
      })
      .links(toRaw(props.linksArray))
    //.distance(50)
    //.strength(0.1)
  );
  simulation.value.force(
    "collide",
    d3.forceCollide().radius(50).iterations(10).strength(0.1)
  );
  simulation.value.alpha(1).restart();
  reDraw();
};

onMounted(() => {
  init();
});
watch(props, () => {
  update();
});
</script>

<template>
  <div className="graph">
    <div className="svgContainer">
      <svg ref="svgRef" width="1280" height="720"></svg>
    </div>
  </div>
</template>
