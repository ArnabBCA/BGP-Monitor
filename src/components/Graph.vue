<script setup>
import * as d3 from "d3";
import { ref, onMounted, watch, toRaw } from "vue";

const svgRef = ref(null);
const simulation = ref(null);
const g_links = ref(null);
const g_nodes = ref(null);
const g_label = ref(null);

const nodes = ref([]);
const links = ref([]);

const props = defineProps({
  nodesArray: Array,
  linksArray: Array,
});

const init = () => {
  const svg = d3.select(svgRef.value);
  svg.call(d3.zoom().on("zoom", zoomed));
  const width = +svg.attr("width");
  const height = +svg.attr("height");

  simulation.value = d3
    .forceSimulation()
    .alphaDecay(0.2)
    .force("charge", d3.forceManyBody().strength(-300))
    .force("center", d3.forceCenter(width / 2, height / 2))
    .on("tick", ticked);

  g_links.value = svg.append("g").attr("class", "links");
  g_nodes.value = svg.append("g").attr("class", "nodes");
  g_label.value = svg.append("g").attr("class", "labels");
};

const reDraw = () => {
  const update_nodes = g_nodes.value.selectAll("circle").data(nodes.value);
  update_nodes
    .enter()
    .append("circle")
    .merge(update_nodes)
    .style("fill", (d) => {
      if (d.isOriginNode) {
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
  update_nodes.exit().remove();

  const update_links = g_links.value.selectAll("line").data(links.value);
  update_links.enter().append("line").merge(update_links);
  update_links.exit().remove();

  const update_labels = g_label.value.selectAll("text").data(nodes.value);
  update_labels
    .enter()
    .append("text")
    .merge(update_labels)
    .text(function (d) {
      return d.name;
    });
  update_labels.exit().remove();
};

const update = () => {
  const newNodes = JSON.parse(JSON.stringify(props.nodesArray));
  const newLinks = JSON.parse(JSON.stringify(props.linksArray));

  newNodes.forEach((n) => {
    // Check if the node with the same id exists in nodes.value
    if (!nodes.value.some((node) => node.id === n.id)) {
      // If it doesn't exist, push it to nodes.value
      nodes.value.push(n);
    }
  });
  // Remove nodes that are not present in props.nodesArray
  nodes.value = nodes.value.filter((node) =>
    newNodes.some((n) => n.id === node.id)
  );

  newLinks.forEach((link) => {
    // Check if the link or its reverse exists in links.value
    if (
      !links.value.some(
        (l) =>
          (l.source === link.source && l.target === link.target) ||
          (l.source === link.target && l.target === link.source)
      )
    ) {
      // If it doesn't exist, push it to links.value
      links.value.push(link);
    }
  });
  // Remove links that are not present in props.linksArray
  links.value = links.value.filter((link) =>
    newLinks.some(
      (l) =>
        (l.source === link.source && l.target === link.target) ||
        (l.source === link.target && l.target === link.source)
    )
  );

  simulation.value.nodes(nodes.value);
  simulation.value.force(
    "link",
    d3
      .forceLink()
      .id(function (d) {
        return d.id;
      })
      .links(links.value)
    //.distance(50)
    //.strength(0.1)
  );
  simulation.value.force(
    "collide",
    d3.forceCollide().radius(50).strength(1)
    .iterations(3)
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
</script>

<template>
  <div className="graph">
    <div className="svgContainer">
      <svg ref="svgRef" width="1280" height="720"></svg>
    </div>
  </div>
</template>
