<script setup>
import { onMounted, onUnmounted, ref, toRaw, watch } from "vue";



const nodes = [
  {
    id: 1,
    name: "A",
  },
  {
    id: 2,
    name: "B",
  },
  {
    id: 3,
    name: "C",
  },
  {
    id: 4,
    name: "D",
  },
  {
    id: 5,
    name: "E",
  },
  {
    id: 6,
    name: "F",
  },
  {
    id: 7,
    name: "G",
  },
  {
    id: 8,
    name: "H",
  },
  {
    id: 9,
    name: "I",
  },
  {
    id: 10,
    name: "J",
  },
];
const links = [
  {
    source: 1,
    target: 2,
  },
  {
    source: 1,
    target: 5,
  },
  {
    source: 1,
    target: 6,
  },
  {
    source: 2,
    target: 3,
  },
  {
    source: 2,
    target: 7,
  },
  {
    source: 3,
    target: 4,
  },
  {
    source: 8,
    target: 3,
  },
  {
    source: 4,
    target: 5,
  },
  {
    source: 4,
    target: 9,
  },
  {
    source: 5,
    target: 10,
  },
];

const isPlaying = ref(false);

//Example Filter parameters
const params = ref({
  peer: "",
  path: "",
  prefix: ["138.121.148.0/22"],
  type: "UPDATE",
  require: "",
  moreSpecific: true,
  lessSpecific: false,
  host: "rrc21",
  socketOptions: {
    includeRaw: false,
    acknowledge: false,
  },
});

//Sending ris message protocol after establishing WebSocket connection.
const sendRisProtocol = (protocol, paramData) => {
  socket.value.send(
    JSON.stringify({
      type: protocol, //Sending the protocol type.
      data: paramData, //Sending the filter parameters.
    })
  );
};

//Changing ris message protocol
const toggleRisProtocol = () => {
  if (!socket.value) {
    connectWebSocket(); //For (RECONNECT) closed due to inactivity.
  } 
  else if (rrcList.value.length === 0) {
    sendRisProtocol("request_rrc_list", null); //Requesting RRC List.
  } 
  else if (isPlaying.value) {
    sendRisProtocol("ris_subscribe", params.value); //Subscribing. (PLAY)
  } 
  else {
    sendRisProtocol("ris_unsubscribe", params.value); //Unsubscribing. (PAUSE)
  }
};

watch(isPlaying, () => {
  toggleRisProtocol(); //For (PLAY/PAUSE).
});

const socket = ref(null);
const rccList = ref([]);
const disbleButton = ref(false);

const hopLimit = ref(3); //Example hopLimit value.

const connectWebSocket = () => {
  /*
    Exisiting code for WebSocket connection.
  */
  socket.value.onmessage = (event) => {
    const res = JSON.parse(event.data);
    if (res.type === "ris_message") {
      messages.value.push(res.data); //Storing all the messages.
      handleFilterMessage(res.data); //Filtering the messages.
    }
    /*
      Exisiting code for handling other "ris_message" types.
    */
  };
};
/*
  Existing code logic
*/


const messages = ref([]);
const rrcList = ref([]);
const disableButton = ref(false);
const socket = ref(null);

const connectWebSocket = () => {
  //Connecting to RIS Live WebSocket API.
  socket.value = new WebSocket(
    "wss://ris-live.ripe.net/v1/ws/?client=js-example-1"
  );
  //Checking if the WebSocket is in CONNECTING state.
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
    } 
    else if (res.type === "ris_rrc_list") {
      rrcList.value = res.data; //Storing the Hostnames for user to select.
    } 
    else if (res.type === "ris_error") {
      console.log("Ris Live Error:", res.data.message); //Handling the ris error.
    }
    //Handle other ris_messages types (optional).
  };
};

onMounted(() => {
  connectWebSocket(); //Initializing connection For Getting the RRC List.
});
onUnmounted(() => {
  socket.value.close(); //Closing the WebSocket connection.
});
</script>

<template>
  <Graph :nodesArray="nodes" :linksArray="links" /> <!-- Passing nodes and links as props -->
</template>
