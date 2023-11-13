<script lang="ts">
  import {
    onMount,
    onDestroy,
    createEventDispatcher,
    afterUpdate,
    setContext,
  } from "svelte";
  import FA2Layout from "graphology-layout-forceatlas2/worker";
  import forceAtlas2 from "graphology-layout-forceatlas2";
  import Graph from "graphology";
  import type { SigmaNodeEventPayload } from "sigma/sigma";
  import type { Attributes } from "graphology-types";
  import type { EdgeDisplayData, NodeDisplayData } from "sigma/types";
  import { Mode, settings, uiVisible, graph } from "./stores";
  import { adamicAdar, coCitation, ResultMap } from "./analysis";
  import {
    shortestPathDirected,
    shortestPathUndirected,
    shortestPathEdgePredicate,
  } from "./shortestPath";
  import { nodeNameIndex } from "./graph";
  import { findNodes } from "./settings/helpers/search";

  import CircleNodeProgram from "sigma/rendering/webgl/programs/node.fast";
  import getNodeProgramImage from "sigma/rendering/webgl/programs/node.image";
  import { getTheme } from "./settings/themes";
  import drawHover from "./sigma/drawHover";
  import { ExtendedSigma } from "./sigma/sigma";

  let sigmaRef: HTMLElement;
  let sigma: ExtendedSigma | undefined;
  let fa2Layout: FA2Layout | undefined;
  export let size: number;

  const dispatch = createEventDispatcher();

  const red = "#f87171";
  const orange = "#fb923c";

  setContext("sigma", {
    getGraph: () => sigma?.getGraph(),
  });

  onMount(async () => {
    const graph = new Graph();
    const theme = getTheme($settings.themeMode);
    sigma = new ExtendedSigma(graph, sigmaRef, {
      allowInvalidContainer: true,
      edgeReducer,
      defaultNodeColor: theme.nodeColor,
      defaultEdgeColor: theme.edgeColor,
      labelColor: {
        color: theme.nodeLabelColor,
      },
      labelBackgroundColor: theme.nodeLabelBackground,
      labelShadowColor: theme.nodeLabelShadowColor,
      nodeProgramClasses: {
        circle: CircleNodeProgram,
        image: getNodeProgramImage(),
      },
      hoverRenderer: drawHover,
    });
    setNodeSizeAndColor(graph);

    sigma.on("clickNode", handleNodeClick);

    if ($settings.cameraState) {
      sigma.getCamera().setState($settings.cameraState);
    }
  });

  $: if (sigma) {
    sigma.setSetting("labelRenderedSizeThreshold", $settings.labelThreshold);
  }

  $: if (!$uiVisible) {
    if (fa2Layout) {
      fa2Layout.stop();
    }
  } else {
    if (fa2Layout) {
      fa2Layout.start();
    }
  }

  afterUpdate(() => {
    if (!sigma) return;
    let defaultSettings = forceAtlas2.inferSettings(size);
    fa2Layout?.kill();
    const graph = sigma.getGraph();
    fa2Layout = new FA2Layout(graph, {
      settings: {
        ...defaultSettings,
        gravity: $settings.nodesGravity * defaultSettings.gravity,
        strongGravityMode: true,
        scalingRatio: $settings.scalingRatio * defaultSettings.scalingRatio,
        barnesHutTheta: 0.2,
        edgeWeightInfluence: $settings.edgeWeightInfluence * -1, // reverse fills more natural
        slowDown: 100,
      },
    });
    fa2Layout.start();
    sigma.refresh();
  });

  onDestroy(async () => {
    if (fa2Layout) {
      fa2Layout.kill();
      fa2Layout = undefined;
    }
    if (sigma) {
      $settings.cameraState = sigma.getCamera().getState();
      sigma.kill();
      sigma = undefined;
    }
  });

  const handleNodeClick = async ({ node }: SigmaNodeEventPayload) => {
    dispatch("nodeclick", node);
  };

  const maxSize = (size: number, max: number) => {
    if (size > max) {
      return max;
    } else {
      return size;
    }
  };

  let shortestNodePath: Array<string> | undefined | null;
  let shortestEdgePath: Array<string> | undefined | null;
  let adamicAdarResults: ResultMap | undefined;
  let nodeIndex: Map<string, string> | undefined;
  let coCitationResults: any;
  let foundNodeIds: Array<string> = [];

  $: if (sigma) {
    findNodes($settings.search, sigma.getGraph()).then((res) => {
      foundNodeIds = res;
    });
  }

  function setNodeSizeAndColor(graph: Graph) {
    graph.updateEachNodeAttributes((node, attr: Attributes) => {
      if ($settings.size === "in") {
        attr.size = maxSize(Math.max(2, graph.neighbors(node).length / 2), 16);
      } else if ($settings.size === "out") {
        attr.size = maxSize(graph.inDegree(node), 16);
      }
    });
  }

  // update style on change settings
  $: {
    if (sigma) {
      const theme = getTheme($settings.themeMode);
      const graph = sigma.getGraph();
      sigma.setSetting("defaultEdgeColor", theme.edgeColor);
      sigma.setSetting("defaultNodeColor", theme.nodeColor);
      sigma.setSetting("labelColor", { color: theme.nodeLabelColor });
      sigma.setSetting("labelBackgroundColor", theme.nodeLabelBackground);
      sigma.setSetting("labelShadowColor", theme.nodeLabelShadowColor);
      graph.updateEachNodeAttributes((_, attr) => {
        attr.color = theme.nodeColor;
        attr.size = 2;
        return attr;
      });
      $settings.filters.forEach((filter) => {
        graph.updateEachNodeAttributes((node, attr: Attributes) => {
          attr.highlighted = false;
          attr.size = (attr.size ?? attr.size) + 2;
          if (
            filter.searchType === "color" &&
            filter.foundNodeIds?.includes(node)
          ) {
            attr.color = filter.searchColor;
            return attr;
          }
          return attr;
        });
      });
      if (foundNodeIds) {
        graph.updateEachNodeAttributes((node, attr: Attributes) => {
          attr.highlighted = false;
          attr.size = (attr.size ?? attr.size) + 2;

          return attr;
        });
      }
      graph.updateEachNodeAttributes((node, attr: Attributes) => {
        if (foundNodeIds?.includes(node)) {
          attr.color = orange;
          attr.highlighted = true;
          attr.size = (attr.size ?? attr.size) + 2;
          return attr;
        }
        setNodeSizeAndColor(attr, node, graph);

        if ($settings.mode === Mode.ShortestPath) {
          const pathA =
            $settings.pathA && nodeIndex?.get($settings.pathA.toUpperCase());
          const pathB =
            $settings.pathB && nodeIndex?.get($settings.pathB.toUpperCase());

          if (
            shortestNodePath?.includes(node) ||
            node === pathA ||
            node === pathB
          ) {
            attr.color = red;
            attr.zIndex = 2;
            attr.highlighted = true;
          } else {
            attr.zIndex = -1;
          }
        }

        if ($settings.mode === Mode.AdamicAdar && adamicAdarResults) {
          const pathA =
            $settings.pathA && nodeIndex?.get($settings.pathA.toUpperCase());
          if (pathA && node === pathA) {
            attr.size = 10;
            attr.zIndex = 2;
            attr.color = orange;
            attr.highlighted = true;
          } else if (adamicAdarResults[node]) {
            attr.color = red;
            attr.size = maxSize(
              $settings.bubbleSize * adamicAdarResults[node].measure,
              32
            );
            attr.label = `${adamicAdarResults[node].measure} ${attr.label}`;
          }
        }
        if ($settings.mode === Mode.CoCitation && coCitationResults) {
          const pathA =
            $settings.pathA && nodeIndex?.get($settings.pathA.toUpperCase());
          if (pathA && node === pathA) {
            attr.size = 10;
            attr.zIndex = 2;
            attr.color = orange;
            attr.highlighted = true;
          } else if (coCitationResults[node]) {
            attr.color = red;
            attr.size = $settings.bubbleSize * coCitationResults[node].measure;
            attr.label = `${coCitationResults[node].measure} ${attr.label}`;
          }
        }
        return attr;
      });
    }
  }

  $: if (sigma && ($settings.pathA || $settings.pathB || $settings.search)) {
    nodeIndex = nodeNameIndex($graph);
  }

  $: if (
    sigma &&
    $settings.mode === Mode.ShortestPath &&
    $settings.pathA &&
    $settings.pathB
  ) {
    $settings.filterLength;
    // paht finding algorithm is not working with hidden nodes
    const graph = sigma.getGraph().copy();
    graph.forEachNode((node, attrs) => {
      if (attrs.hidden) {
        graph.dropNode(node);
      }
    });
    const results = $settings.directed
      ? shortestPathDirected(graph, $settings.pathA, $settings.pathB)
      : shortestPathUndirected(graph, $settings.pathB, $settings.pathA);
    shortestNodePath = results.nodes;
    shortestEdgePath = results.edges;
  } else {
    shortestNodePath = undefined;
    shortestEdgePath = undefined;
  }

  $: if (sigma && $settings.mode === Mode.AdamicAdar && $settings.pathA) {
    const graph = sigma.getGraph();
    const pathA =
      $settings.pathA && nodeIndex?.get($settings.pathA.toUpperCase());
    if (pathA) {
      adamicAdarResults = adamicAdar(graph, pathA);
      console.log(adamicAdarResults);
    } else {
      adamicAdarResults = undefined;
    }
  } else {
    adamicAdarResults = undefined;
  }

  $: if (sigma && $settings.mode === Mode.CoCitation && $settings.pathA) {
    const pathA = nodeIndex?.get($settings.pathA.toUpperCase());
    if (pathA) {
      (async () =>
        (coCitationResults = await coCitation(sigma.getGraph(), pathA)))();
    } else {
      coCitationResults = undefined;
    }
  } else {
    coCitationResults = undefined;
  }

  const edgeReducer = (edge: string, data: Attributes) => {
    const res: Partial<EdgeDisplayData> = { ...data };
    if (
      sigma &&
      $settings.mode === Mode.ShortestPath &&
      shortestPathEdgePredicate(shortestEdgePath, edge)
    ) {
      res.color = red;
      res.size = 5;
    }
    return res;
  };
</script>

<div class="sigma" bind:this={sigmaRef} />
<slot />

<style>
  .sigma {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
    overflow: hidden;
  }
</style>
