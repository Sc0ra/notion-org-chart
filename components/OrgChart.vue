<template>
  <div class="tree-container" ref="container">
    <svg class="svg vue-tree" ref="svg" :style="initialTransformStyle"></svg>

    <div
      class="dom-container"
      ref="domContainer"
      :style="initialTransformStyle"
    >
      <transition-group name="tree-node-item" tag="div">
        <div
          class="node-slot"
          v-for="(node, index) of nodeDataList"
          @click="onClickNode(index)"
          :key="node.data.value._key || index"
          :style="{
            left: `${node.x}px`,
            top: `${node.y}px`,
            width: `${node.data.value.nodeWidth}px`,
            height: `${node.data.value.nodeHeight}px`,
          }"
        >
          <slot
            name="node"
            v-bind:node="node.data"
            v-bind:collapsed="node.data.value._collapsed"
          >
            <span>{{ node.data.value.label }}</span>
          </slot>
        </div>
      </transition-group>
    </div>
  </div>
</template>

<script setup lang="ts">
import * as d3 from "d3";
import { HierarchyPointLink, HierarchyPointNode } from "d3";
import { v4 as uuidv4 } from "uuid";

enum LinkStyle {
  CURVE = "CURVE",
  STRAIGHT = "STRAIGHT",
}

interface Datum {
  [key: string]: any;
}

type Tree<T> = {
  value: T;
  children?: Tree<T>[] | null;
};

type Data = Tree<Datum>;

const MATCH_TRANSLATE_REGEX = /translate\((-?\d+)px, ?(-?\d+)px\)/i;

const initTransformX = ref(0);
const initTransformY = ref(0);
const linkStyle: LinkStyle = LinkStyle.STRAIGHT;

const DEFAULT_NODE_WIDTH = 100;
const DEFAULT_NODE_HEIGHT = 100;
const DEFAULT_LEVEL_HEIGHT = 200;

const ANIMATION_DURATION = 800;

const config = {
  nodeWidth: DEFAULT_NODE_WIDTH,
  nodeHeight: DEFAULT_NODE_HEIGHT,
  levelHeight: DEFAULT_LEVEL_HEIGHT,
  collapseEnabled: true,
};

const initialTransformStyle = computed(() => ({
  transform: `scale(1) translate(${initTransformX.value}px, ${initTransformY.value}px)`,
  transformOrigin: "center",
}));

const dataset: Data = {
  value: {
    label: "1",
  },
  children: [
    {
      value: { label: "2" },
      children: [{ value: { label: "3" } }, { value: { label: "4" } }],
    },
    { value: { label: "5" } },
  ],
};
let nodeDataList = ref<HierarchyPointNode<Data>[]>([]);
let linkDataList = ref<HierarchyPointLink<Data>[]>([]);
let currentScale = 1;

const _dataset = computed(() => updatedInternalData(dataset));
watch(
  () => _dataset.value,
  () => {
    draw();
    initTransform();
  },
  { deep: true }
);

const domContainer = ref<HTMLDivElement>();
const svg = ref<HTMLDivElement>();
const container = ref<HTMLDivElement>();

onMounted(() => {
  init();
});

function init() {
  draw();
  enableDrag();
  initTransform();
}

function draw() {
  var initialTree = buildTree(_dataset.value);
  linkDataList.value = initialTree.links;
  nodeDataList.value = initialTree.nodes;
  // Do not render the invisible root node.
  nodeDataList.value.splice(0, 1);
  linkDataList.value = linkDataList.value.filter(
    (x: any) => x.source.data.value.name !== "__invisible_root"
  );
  const svgRef = d3.select(svg.value as HTMLDivElement);
  const links = svgRef.selectAll(".link").data(linkDataList.value, (d: any) => {
    return `${d.source.data.value._key}-${d.target.data.value._key}`;
  });
  links
    .enter()
    .append("path")
    .style("opacity", 0)
    .transition()
    .duration(ANIMATION_DURATION)
    .ease(d3.easeCubicInOut)
    .style("opacity", 1)
    .attr("class", "link")
    .attr("d", function (d, i) {
      return generateLinkPath(d) as any;
    });
  links
    .transition()
    .duration(ANIMATION_DURATION)
    .ease(d3.easeCubicInOut)
    .attr("d", function (d) {
      return generateLinkPath(d) as any;
    });
  links
    .exit()
    .transition()
    .duration(ANIMATION_DURATION / 2)
    .ease(d3.easeCubicInOut)
    .style("opacity", 0)
    .remove();
}

function buildTree(rootNode: Data): {
  nodes: HierarchyPointNode<Data>[];
  links: HierarchyPointLink<Data>[];
} {
  const treeBuilder = d3
    .tree<Data>()
    .nodeSize([config.nodeWidth, config.levelHeight]);
  const tree = treeBuilder(d3.hierarchy(rootNode));
  return {
    nodes: tree.descendants(),
    links: tree.links(),
  };
}

function generateLinkPath(d: any): string | null {
  switch (linkStyle) {
    case LinkStyle.CURVE: {
      const linkPath = d3.linkVertical();
      linkPath
        .x((d: any) => d.x)
        .y((d: any) => d.y)
        .source(
          (d: any) =>
            ({
              x: d.source.x,
              y: d.source.y,
            } as any)
        )
        .target(
          (d: any) =>
            ({
              x: d.target.x,
              y: d.target.y,
            } as any)
        );
      return linkPath(d);
    }
    case LinkStyle.STRAIGHT: {
      const linkPath = d3.path();
      let sourcePoint = { x: d.source.x, y: d.source.y };
      let targetPoint = { x: d.target.x, y: d.target.y };
      const secondPoint = { x: sourcePoint.x, y: targetPoint.y };
      linkPath.moveTo(sourcePoint.x, sourcePoint.y);
      linkPath.lineTo(secondPoint.x, secondPoint.y);
      linkPath.lineTo(targetPoint.x, targetPoint.y);
      return linkPath.toString();
    }
  }
}

function updatedInternalData(externalData?: Data): Data {
  var data: Data = { value: { name: "__invisible_root" }, children: [] };
  if (!externalData) return data;
  if (Array.isArray(externalData)) {
    for (var i = externalData.length - 1; i >= 0; i--) {
      data.children?.push(deepCopy(externalData[i]));
    }
  } else {
    data.children?.push(deepCopy(externalData));
  }
  return data;
}

function deepCopy(node: Data) {
  let obj: Data = { value: { _key: uuidv4(), ...node.value } };
  if (node.children) {
    obj.children = [
      ...node.children.map((childrenNode) => deepCopy(childrenNode)),
    ];
  }
  return obj;
}

function enableDrag() {
  const svgElement = svg.value as HTMLDivElement;
  const containerRef = container.value as HTMLDivElement;
  let startX = 0;
  let startY = 0;
  let isDrag = false;
  let mouseDownTransform = "";
  containerRef.onmousedown = (event: any) => {
    mouseDownTransform = svgElement.style.transform;
    startX = event.clientX;
    startY = event.clientY;
    isDrag = true;
  };
  containerRef.onmousemove = (event: any) => {
    if (!isDrag) return;
    const originTransform = mouseDownTransform;
    let originOffsetX = 0;
    let originOffsetY = 0;
    if (originTransform) {
      const result = originTransform.match(MATCH_TRANSLATE_REGEX);
      if (result !== null && result.length !== 0) {
        const [offsetX, offsetY] = result.slice(1);
        originOffsetX = parseInt(offsetX);
        originOffsetY = parseInt(offsetY);
      }
    }
    let newX =
      Math.floor((event.clientX - startX) / currentScale) + originOffsetX;
    let newY =
      Math.floor((event.clientY - startY) / currentScale) + originOffsetY;
    let transformStr = `translate(${newX}px, ${newY}px)`;
    if (originTransform) {
      transformStr = originTransform.replace(
        MATCH_TRANSLATE_REGEX,
        transformStr
      );
    }
    svgElement.style.transform = transformStr;
    (domContainer.value as HTMLDivElement).style.transform = transformStr;
  };
  containerRef.onmouseup = (event: any) => {
    startX = 0;
    startY = 0;
    isDrag = false;
  };
}

function initTransform() {
  const containerWidth = (container.value as HTMLDivElement).offsetWidth;
  initTransformX.value = Math.floor(containerWidth / 2);
  initTransformY.value = 0;
}

function onClickNode(index: any) {
  if (config.collapseEnabled) {
    const curNode = nodeDataList.value[index];
    if (curNode.data.children) {
      curNode.data.value._children = curNode.data.children;
      curNode.data.children = null;
      curNode.data.value._collapsed = true;
    } else {
      curNode.data.children = curNode.data.value._children;
      curNode.data.value._children = null;
      curNode.data.value._collapsed = false;
    }
    draw();
  }
}
</script>

<style lang="scss">
.tree-container {
  .node {
    fill: grey !important;
  }
  .link {
    stroke-width: 2px !important;
    fill: transparent !important;
    stroke: #cecece !important;
  }
}
</style>

<style lang="scss" scoped>
.tree-node-item-enter,
.tree-node-item-leave-to {
  transition-timing-function: ease-in-out;
  transition: transform 0.8s;
  opacity: 0;
}
.tree-node-item-enter-active,
.tree-node-item-leave-active {
  transition-timing-function: ease-in-out;
  transition: all 0.8s;
}
.tree-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  .vue-tree {
    position: relative;
  }
  > svg,
  .dom-container {
    width: 100%;
    height: 100%;
    position: absolute;
    left: 0;
    top: 0;
    overflow: visible;
    transform-origin: 0 50%;
  }
  .dom-container {
    z-index: 1;
    pointer-events: none;
  }
}
.node-slot {
  cursor: pointer;
  pointer-events: all;
  position: absolute;
  background-color: transparent;
  box-sizing: border-box;
  transform: translate(-50%, -50%);
  display: flex;
  align-items: center;
  justify-content: center;
  box-sizing: content-box;
  transition: all 0.8s;
  transition-timing-function: ease-in-out;
}
</style>
