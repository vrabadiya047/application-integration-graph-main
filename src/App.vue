<script setup>
import "@leanix/reporting";
import { reactive, ref, unref, watch } from "vue";
import * as vNG from "v-network-graph";
import { ForceLayout } from "v-network-graph/lib/force-layout";

// Initialize reactive variables
const factSheetTypes = ref([]);
const selectedFactSheetType = ref(null);
const applications = ref([]);
// const interfaceVa = ref([])
const zoom = ref(1);
const graph = ref(null);
const justInterface = ref([]);
const nodeColors = reactive({});

const queryConsumerApplicationToInterface = `relConsumerApplicationToInterface {
            edges {
              node {
                factSheet {
                  id
                  name
                }
              }
            }
          }`;

const queryProviderApplicationToInterface = `relProviderApplicationToInterface {
  edges {
    node {
      factSheet {
        id
        name
      }
    }
  }
}`;

const initializeReport = async () => {
  const setup = await lx.init();
  factSheetTypes.value = Object.keys(setup.settings.dataModel.factSheets);
  selectedFactSheetType.value = factSheetTypes.value?.[0] || null;

  const config = {
    menuActions: {
      showConfigure: true,
      configureCallback: openReportConfigurationModal,
    },
    facets: [
      {
        key: "main",
        fixedFactSheetType: "Application",
        attributes: [
          "id",
          "name",
          queryConsumerApplicationToInterface,
          queryProviderApplicationToInterface,
        ],
        callback: MyFunction.bind(this),
      },
    ],
    ui: {
      update: (selection) => {
        const prevZoom = zoom.value;
        zoom.value = selection.elements.values.zoom;

        if (zoom.value < prevZoom) {
          unref(graph)?.zoomOut();
        } else if (zoom.value > prevZoom) {
          unref(graph)?.zoomIn();
        }
      },

      elements: {
        root: {
          items: [
            {
              id: "zoom",
              type: "zoom",
            },
          ],
        },
        values: {
          zoom: unref(zoom),
        },
      },
    },

    reportViewFactSheetType: "Application",
    reportViewCallback: function (data) {
      // Build map from legendItem id to the item itself
      var legendItemMap = {};
      data.legendItems.forEach((item) => {
        legendItemMap[item.id] = item;
      });

      // Create object to store legend IDs and colors
      const legendColors = {};
      // Print mappings
      data.mapping.forEach((fsMapping) => {
        const legendItem = legendItemMap[fsMapping.legendId];
        legendColors[fsMapping.fsId] = legendItem.bgColor;
      });
      // store legendColors object in nodeColors reactive variable
      nodeColors.value = legendColors;
      //console.log(legendColors)
    },
  };

  lx.ready(config);
};

const MyFunction = (data) => {
  applications.value = data;
  let countApplication = 0;
  applications.value.forEach((elements) => {
    //Only Application with relation to Interfaces
    if (
      elements.relConsumerApplicationToInterface.edges.length != 0 ||
      elements.relProviderApplicationToInterface.edges.length != 0
    ) {
      countApplication += 1;
    }
  });

  let countConsumer = 0;
  applications.value.forEach((elements) => {
    //GET all Consumer app to Interfaces
    if (elements.relConsumerApplicationToInterface.edges.length != 0) {
      let testingVariable;
      testingVariable = elements.relConsumerApplicationToInterface.edges.map(
        ({ node }) => node.factSheet.name
      );
      for (let i = 0; i < testingVariable.length; i++) {
        /* empty */
      }
      countConsumer += 1;
    }
  });

  //GET all Provider app To Interface
  let countProvider = 0;
  applications.value.forEach((el) => {
    if (el.relProviderApplicationToInterface.edges.length != 0) {
      let getName;
      getName = el.relProviderApplicationToInterface.edges.map(
        ({ node }) => node.factSheet.name
      );
      for (let i = 0; i < getName.length; i++) {
        countProvider += 1;
      }
    }
  });
  //countAppWithProviderInterface.value = countProvider
};

const openReportConfigurationModal = async () => {
  const fields = {
    factSheetType: {
      type: "SingleSelect",
      label: "FactSheet Type",
      options: factSheetTypes.value.map((factSheetType) => ({
        value: factSheetType,
        label: lx.translateFactSheetType(factSheetType),
      })),
    },
  };
  const initialValues = { factSheetType: selectedFactSheetType.value };
  const values = await lx.openFormModal(fields, initialValues);
  if (values) selectedFactSheetType.value = values.factSheetType;
};

const fetchGraphQLData = async () => {
  const interfaceComponents = `{
  allFactSheets(factSheetType: Interface) {
    totalCount
    edges {
      node {
        id
        name
        ... on Interface{
          relInterfaceToDataObject{
            totalCount
            edges {
  
      node {
        
        factSheet{
          name
          id
        }
          }
        }
        }
        }
      }
    }}}
  `;
  try {
    lx.showSpinner();
    //JUST INTERFACE
    justInterface.value = await lx
      .executeGraphQL(interfaceComponents)
      .then((data) => data.allFactSheets.edges.map(({ node }) => node));
  } finally {
    lx.hideSpinner();
  }
};

async function fetchDataObjectsForInterface(interfaceId) {
  try {
    // Construct a GraphQL query to fetch data objects related to the interface
    const query = `
      query FetchDataObjectsForInterface($interfaceId: ID!) {
        allFactSheets(
          filter: {
            id: {
              eq: $interfaceId
            }
          }
        ) {
          edges {
            node {
              id
              name
              ... on Interface {
                relInterfaceToDataObject {
                  totalCount
                  edges {
                    node {
                      factSheet {
                        name
                        id
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    `;

    // Define the variables for the GraphQL query
    const variables = {
      interfaceId: interfaceId,
    };

    // Make a GraphQL request to fetch data objects related to the interface
    const response = await lx.executeGraphQL(query, variables);

    // Extract and return the data objects from the response
    const dataObjects =
      response.allFactSheets.edges[0].node.relInterfaceToDataObject.edges.map(
        ({ node }) => node.factSheet
      );
    return dataObjects;
  } catch (error) {
    console.error("Error fetching data objects for the interface:", error);
    throw error;
  }
}

const getNodes = () => {
  const nodes = {};
  const nodeName = new Set();
  const interfaceCounts = [];

  // Collect all node names and their corresponding IDs
  applications.value.forEach((elements) => {
    if (
      elements.relConsumerApplicationToInterface.edges.length !== 0 ||
      elements.relProviderApplicationToInterface.edges.length !== 0
    ) {
      const nodeId = elements.id.trim(); // Use the ID as the node ID
      const nodeNameValue = elements.name.trim();
      nodeName.add([nodeNameValue, nodeId]);

      // Collect the number of interfaces for this node
      const interfaceCount =
        elements.relConsumerApplicationToInterface.edges.length +
        elements.relProviderApplicationToInterface.edges.length;
      interfaceCounts.push(interfaceCount);
    }
  });

  // Find the maximum number of interfaces across all nodes
  const maxInterfaceCount = Math.max(...interfaceCounts);

  // Create an object to map legend IDs to node names
  const legendIdToNodeName = {};
  nodeName.forEach((node) => {
    legendIdToNodeName[node[1]] = node[0];
  });

  // Set the size of each node based on the maximum interface count
  nodeName.forEach((nodeName) => {
    const interfaceCount = applications.value.reduce((total, app) => {
      if (app.name.trim() === nodeName[0]) {
        total +=
          app.relConsumerApplicationToInterface.edges.length +
          app.relProviderApplicationToInterface.edges.length;
      }
      return total;
    }, 0);

    const nodeNameValue = nodeName[0]; // Get the node name
    const nodeId = nodeName[1]; // Get the node ID

    nodes[nodeId] = {
      // Use the ID as the node key
      id: nodeId,
      name: nodeNameValue,
      size: Math.max(10, Math.round((interfaceCount / maxInterfaceCount) * 25)),
      color: nodeColors.value[nodeId] || "#FFFFFF",
      interfaceCount: interfaceCount,
    };
  });

  return nodes;
};

const getEdges = () => {
  const edges = {};
  let edgeId = 1;
  const edgesObject = reactive(edges);

  // Create edges between applications based on shared consumer or provider interfaces
  applications.value.forEach((app1) => {
    applications.value.forEach((app2) => {
      if (app1.id === app2.id) return; // Skip self-connections

      // Check for shared consumer interfaces
      const sharedConsumerInterfaces =
        app1.relConsumerApplicationToInterface.edges.filter(
          ({ node: { factSheet: consumerInterface } }) => {
            return app2.relProviderApplicationToInterface.edges.some(
              ({ node: { factSheet: providerInterface } }) => {
                return (
                  providerInterface.name.split(">>")[0].trim() ===
                  consumerInterface.name.split(">>")[0].trim()
                );
              }
            );
          }
        );

      // Check for shared provider interfaces
      const sharedProviderInterfaces =
        app1.relProviderApplicationToInterface.edges.filter(
          ({ node: { factSheet: providerInterface } }) => {
            return app2.relConsumerApplicationToInterface.edges.some(
              ({ node: { factSheet: consumerInterface } }) => {
                return (
                  consumerInterface.name.split(">>")[0].trim() ===
                  providerInterface.name.split(">>")[0].trim()
                );
              }
            );
          }
        );

      // Create edge if shared interfaces exist
      if (
        sharedConsumerInterfaces.length > 0 ||
        sharedProviderInterfaces.length > 0
      ) {
        const edge = {
          source: app1.id.trim(),
          target: app2.id.trim(),
          width: Math.max(
            sharedConsumerInterfaces.length,
            sharedProviderInterfaces.length
          ),
          interfaces: [
            ...sharedConsumerInterfaces.map(({ node }) => node.factSheet.name),
            ...sharedProviderInterfaces.map(({ node }) => node.factSheet.name),
          ],
          interfaceIds: [
            ...sharedConsumerInterfaces.map(({ node }) => node.factSheet.id),
            ...sharedProviderInterfaces.map(({ node }) => node.factSheet.id),
          ],
        };
        edgesObject[`edge${edgeId}`] = edge;
        edgeId++;
      }
    });
  });

  // Remove duplicate edges
  const uniqueEdges = {};
  for (const edgeId in edgesObject) {
    const edge = edgesObject[edgeId];
    const reversedEdge = {
      source: edge.target,
      target: edge.source,
    };
    if (
      !uniqueEdges[edgeId] &&
      !uniqueEdges[`edge${edgeId - 1}`] &&
      !uniqueEdges[`${edge.source}-${edge.target}`] &&
      !uniqueEdges[`${reversedEdge.source}-${reversedEdge.target}`]
    ) {
      uniqueEdges[`${edge.source}-${edge.target}`] = edge;
    }
  }

  return uniqueEdges;
};

const eventHandlers = {
  "node:click": (event) => {
    const nodeId = event.node;

    // Open the side pane and display the consumer and provider interfaces
    lx.openSidePane({
      factSheet: {
        type: "FactSheet",
        factSheetType: "Application",
        factSheetId: nodeId,
        relations: [
          {
            name: "relConsumerApplicationToInterface",
          },
          {
            name: "relProviderApplicationToInterface",
          },
          {
            name: "relApplicationToDataObject",
          },
        ],

        detailFields: ["functionalSuitability", "technicalSuitability"],
        pointOfView: {
          id: "node",
          changeSet: { type: "dateOnly", date: new Date() },
        },
      },
    });
  },
  "edge:click": (event) => {
    const edgeId = event.edge;

    // Get the names and IDs of the interfaces associated with the clicked edge
    const edge = getEdges()[edgeId];
    const interfaceNames = edge.interfaces;
    const interfaceIds = edge.interfaceIds;

    // Map each interface to a separate fact sheet using the original ID
    const items = interfaceNames.map((name, index) => {
      return {
        factSheet: {
          displayName: name,
          type: "Interface",
          id: interfaceIds[index],
        },
      };
    });

    // Open the side pane and display the interface names
    lx.openSidePane({
      factSheet: {
        type: "FactSheetTable",
        label: "Interface(s) (count: " + interfaceNames.length + ")",
        items: items,
      },
    });
  },
};

const getConfigs = () => {
  const configs = vNG.defineConfigs({
    view: {
      fit: true,
      layoutHandler: new ForceLayout({}),
    },
    node: {
      selectable: true,
      // for bubble
      normal: {
        radius: (node) => node.size, // set radius to node size
        color: (node) => node.color,
        strokeWidth: 1,
        strokeColor: "#D3D3D3",
      },

      hover: {
        // while take cursor on bubble
        color: "#c3ebed",
      },
      label: {
        // font color and size
        fontSize: 12,
        color: "#000000",
        fontStyle: "italic",
      },
    },

    edge: {
      selectable: true,

      normal: {
        // line between bubbles
        width: (edge) => edge.width * 0.67, // adjust scaling factor as needed
        color: "#02AFA4",
      },
      hover: {
        // when cursor comes on line
        width: (edge) => edge.width * 2, // adjust scaling factor as needed
        color: "#02AFA4",
      },
    },
  });
  return configs;
};




const Provider = "CTI Consulting GmbH ";
const Support = "info@cti-consulting.de ";
const providerLink = "https://cti.consulting/";
const supportLink = "mailto:info@cti-consulting.de";
// Company logo URL
const companyLogo =
  "https://www.it-nordhessen.net/wp-content/uploads/2023/01/CTI-Consulting_Farbe.jpg";

// Function to handle opening the provider link in a new tab
const openProviderLink = () => {
  window.open(providerLink, "_blank");
};

// Function to handle opening the support link in a new tab
const openSupportLink = () => {
  window.open(supportLink, "_blank");
};

// export png and pdf


// watcher that will trigger the fetchGraphQLData on every selectedFactSheetType update
watch(selectedFactSheetType, fetchGraphQLData);
// watcher that will trigger the updateChart method on every averageCompletion variable update
// we call our report initialization method here...
initializeReport();
</script>

<template>
  <div class="graph-container">
    
    <v-network-graph
      :zoom-level="1.0"
      :nodes="getNodes()"
      :edges="getEdges()"
      :configs="getConfigs()"
      :event-handlers="eventHandlers"
      ref="graph"
      class="visualization"
    />
  </div>
  <svg class="w-4 h-4" fill="none" stroke="white" viewBox="0 0 24 24"  xmlns="http://www.w3.org/2000/svg">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="10" d="M7 21h10a2 2 0 002-2V9.414a1 1 0 00-.293-.707l-5.414-5.414A1 1 0 0012.586 3H7a2 2 0 00-2 2v14a2 2 0 002 2z"></path>
  </svg>
  
</template>
<style>

.visualization {
  width: 100%;
  height: 98.5vh;
  border: 1px solid #000;
}
</style>
