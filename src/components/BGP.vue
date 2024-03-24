<script setup>
import { onMounted, onUnmounted, ref, toRaw, watch } from "vue";
import Graph from "./Graph.vue";
import Sankey from "./Sankey.vue";
import Plotly from "plotly.js-dist";

const filteredMessages = ref([]);
const messages = ref([]);
const rrcList = ref([]);

const hopLimit = ref(3);
const prefixInput = ref("");

const isPlaying = ref(false);
const socket = ref(null);
const disableButton = ref(false);

const nodes = ref([]);
const links = ref([]);

const plot = ref(null);
const count = ref(0);

const peerDetails = ref([]);

const params = ref({
  peer: "",
  path: "",
  prefix: ["170.238.225.0/24"], //"138.121.148.0/22"
  type: "UPDATE",
  require: "",
  moreSpecific: true,
  lessSpecific: false,
  host: "",
  socketOptions: {
    includeRaw: false,
    acknowledge: false,
  },
});

const sendSocketType = (protocol, paramData) => {
  socket.value.send(
    JSON.stringify({
      type: protocol,
      data: paramData,
    })
  );
};

const toggleRisProtocol = () => {
  if (!socket.value) {
    //For (RECONNECT) closed by the server due to inactivity.
    connectWebSocket();
  } else if (rrcList.value.length === 0) {
    //Requesting RRC List.
    console.log("Requesting RRC List");
    sendSocketType("request_rrc_list", null);
  } else if (isPlaying.value) {
    //Subscribing. (PLAY)
    sendSocketType("ris_subscribe", params.value);
    plotMessageLineChart();
  } else {
    //Unsubscribing. (PAUSE)
    sendSocketType("ris_unsubscribe", params.value);
    stopRequested = true;
  }
};

watch(isPlaying, () => {
  toggleRisProtocol();
});
const connectWebSocket = () => {
  //Connecting to RIS Live WebSocket API.
  socket.value = new WebSocket(
    "wss://ris-live.ripe.net/v1/ws/?client=js-example-1"
  );
  //Checking if the WebSocket is connecting.
  if (socket.value.readyState === WebSocket.CONNECTING) {
    disableButton.value = true; //Disabling the PLAY/PAUSE button.
  }
  //Connecton established
  socket.value.onopen = () => {
    disableButton.value = false; //Enabling the PLAY/PAUSE button.
    toggleRisProtocol(); //Toggling the RIS subscription.
  };
  //Disconnected from WebSocket.
  socket.value.onclose = () => {
    socket.value = null; //Setting the socket to null to reconnect.
  };
  //Handling any WebSocket error.
  socket.value.onerror = (error) => {
    console.log("WebSocket Error:", error);
  };
  //Handling different types of WebSocket ris_messages.
  socket.value.onmessage = (event) => {
    const res = JSON.parse(event.data);
    if (res.type === "ris_message") {
      messages.value.push(res.data); //Storing all the messages.
      handleFilterMessages(res.data); //Filtering the messages.
      count.value++;
    } else if (res.type === "ris_rrc_list") {
      rrcList.value = res.data;
    } else if (res.type === "ris_error") {
      console.log("Ris Live Error:", res.data.message);
    }
    /*const peer = res.data.peer;
    const existingPeerIndex = peerDetails.value.findIndex(
      (item) => item.peer === peer
    );
    if (existingPeerIndex !== -1) {
      peerDetails.value[existingPeerIndex].count++;
    } else {
      peerDetails.value.push({ peer, count: 1 });
    }*/
    //console.log(res.data);
  };
};

//Getting a single message as a parameter.
const handleFilterMessages = (data) => {
  // Storing unique different peers messages.
  const uniquePeerMessages = filteredMessages.value.filter(
    (message) => message.peer !== data.peer
  );
  // Adding "uniquePeerMessage" messages and latest message "data".
  filteredMessages.value = [...uniquePeerMessages, data];
};

const graphData = () => {
  // Using arrays to store nodes and links.
  const tempLinks = [];
  const tempNodes = [];

  filteredMessages.value.forEach((message) => {
    // Checking if the message has a path and not empty.
    if (message.path?.length > 0) {
      // Maximum number of hops to be displayed. +1 for the origin node.
      const path = message.path.slice(-(hopLimit.value + 1));
      path.forEach((n, i) => {
        // Check if the node already exists
        if (!tempNodes.some((node) => node.id === n)) {
          // D3.js requires id. We can pass other optional properties as well.
          tempNodes.push({id: n, name: n, isOriginNode: i === path.length - 1});
        }
        if (i < path.length - 1) {
          // -1 to avoid the last element.
          const source = path[i];
          const target = path[i + 1];
          if (
            !tempLinks.some(
              (link) =>
                (link.source === source && link.target === target) ||
                (link.source === target && link.target === source)
            )
          ) {
            // D3.js requires source and target for links.
            tempLinks.push({ source: source, target: target, value: 1 });
          }
        }
      });
    }
  });
  nodes.value = tempNodes;
  links.value = tempLinks;
};
watch([filteredMessages, hopLimit], () => {
  graphData();
});
onMounted(() => {
  connectWebSocket();
});

onUnmounted(() => {
  socket.value.close();
});

watch(rrcList, () => {
  if (rrcList.value.length > 0) {
    params.value.host = rrcList.value[0].split(".")[0];
  }
});

let c = 0;
let stopRequested = false;
const plotMessageLineChart = () => {
  const messageInterval = setInterval(() => {
    const xData = [new Date().toLocaleTimeString()];
    const yData = [count.value];
    Plotly.extendTraces(plot.value, { x: [xData], y: [yData] }, [0])
      .then(() => {
        count.value = 0;
        c++;
      })
      .catch((error) => {
        console.error("Error extending traces:", error);
      });
    if (c > 20) {
      Plotly.relayout(plot.value, {
        xaxis: {
          range: [c - 20, c],
        },
      });
    }
    if (stopRequested && count.value === 0) {
      clearInterval(messageInterval);
      stopRequested = false;
    }
  }, 1000);
};

const handlePlotlyClick = (event) => {
  const point = event.points[0];
  console.log(point);
};

const chartData = ref({
  x: [],
  y: [],
  type: "line",
  line: { shape: "linear" },
  marker: { size: 8 },
});
const chartLayout = ref({
  title: "Messages Received Over Time",
  xaxis: { title: "Timestamp" },
  yaxis: { title: "New Messages", rangemode: "tozero" },
});

const handleSelectedHost = (event) => {
  params.value.host = event.target.value.split(".")[0];
};
const handleHopLimit = (event) => {
  hopLimit.value = parseInt(event.target.value);
};
const handlePrefix = () => {
  const prefix = prefixInput.value.trim();
  if (prefix !== "") {
    params.value.prefix.push(prefix);
    prefixInput.value = "";
  }
};

/*watch(isConnected, () => {
  if (isConnected.value) {
    play();
    plotMessageLineChart();
  } else {
    pause();
    stopRequested = true;
  }
});*/

onMounted(() => {
  //connectWebSocket();
  Plotly.newPlot(plot.value, [chartData.value], chartLayout.value, {
    responsive: true,
  });

  plot.value.on("plotly_click", handlePlotlyClick);
});
</script>

<template>
  <div class="inputContainer">
    <h1>Live BGP messages</h1>
    <div class="messagesContainer">
      <div class="outputMessages">
        <h3>AS Path</h3>
        <h3>Peer</h3>
        <!--<span>Timestamp</span>-->
      </div>
      <div v-for="message in filteredMessages" :key="message.id">
        <div class="outputMessages">
          <!-- v-if="message.data.path?.length > 0" -->
          <span>{{ message.path }}</span>
          <span>{{ message.peer }}</span>
          <!--<span>{{ new Date(message.timestamp * 1000).toLocaleString() }}</span>-->
        </div>
      </div>
    </div>
    <div class="filtersContainer">
      <div class="input">
        <label for="Prefix">Prefix:</label>
        <input
          id="prefix"
          type="text"
          v-model="prefixInput"
          placeholder="IPv4/v6 CIDR Prefix"
        />
        <button @click="handlePrefix" :disabled="prefixInput.trim() === ''">
          Add Prefix
        </button>
        <div class="selectedPrefixList">
          <h3>Selected Prefix</h3>
          <span v-for="prefix in params.prefix" :key="prefix">
            {{ prefix }}
          </span>
        </div>
      </div>
    </div>
    <div class="filtersContainer">
      <button @click="isPlaying = !isPlaying" :disabled="disableButton">
        {{ disableButton ? "Connecting..." : isPlaying ? "Pause" : "Play" }}
      </button>
      <div class="input">
        <label for="rrc">Host Name:</label>
        <select name="rrc" id="rrc" @click="handleSelectedHost">
          <option v-for="rrc in rrcList" :key="rrc" :value="rrc">
            {{ rrc.split(".")[0] }}
          </option>
        </select>
      </div>
      <div class="input">
        <label for="hop">Max Hops: {{ hopLimit }}</label>
        <input
          name="hop"
          id="hop"
          type="range"
          min="1"
          max="9"
          v-model="hopLimit"
          style="padding: 0px"
          @input="handleHopLimit"
        />
      </div>
    </div>
    <Graph :nodesArray="nodes" :linksArray="links" />
    <Sankey :nodesArray="nodes" :linksArray="links" />
    <div class="chart">
      <div ref="plot"></div>
    </div>
  </div>
</template>
