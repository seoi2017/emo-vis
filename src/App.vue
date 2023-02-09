<template>
  <n-card class="container" :title="`EMO Algorithm Iteration - ${algorithm} on ${problem}`" :bordered="false">
    <n-space vertical>
      <n-card>
        <div class="chart"></div>
      </n-card>
      <n-card>
        <div class="graph"></div>
      </n-card>
      <n-slider :value="value" @update:value="update" :step="step" placement="bottom"
        :marks="{ [value[0]]: value[0], [value[1]]: value[1] }" range />
    </n-space>
  </n-card>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import * as d3 from 'd3';
import _ from 'lodash';
import Data from './Data.json';

const value = ref([100, 100]);
const step = ref(10);
const algorithm = ref(Data.ID.algorithm);
const problem = ref(Data.ID.problem);

const xScale = d3.scaleLinear()
  .domain([-50, 50])
  .range([0, 700]);
const yScale = d3.scaleLinear()
  .domain([-50, 50])
  .range([700, 0]);
const xScale2 = d3.scaleLinear()
  .domain([0, 100])
  .range([0, 700]);
const yScale2 = d3.scaleLinear()
  .domain([0, 1])
  .range([50, 0]);
const color = d3.interpolate(d3.rgb(255, 255, 0, 0.5), d3.rgb(0, 176, 80, 0.5));
const area = d3.area()
  .curve(d3.curveLinear)
  .x(it => xScale2(it))
  .y0(() => yScale2(0))
  .y1(it => yScale2(_.find(res, d => d.iteration === it).pfRate));
const res = _.concat(_.map(
  _.range(0, 100, 10),
  it => {
    const objData = _.times(100, id => ({
      "ID": id,
      "dominanceMark": _.random(0, 1),
      "x": _.random(-40, 40, true),
      "y": _.random(-40, 40, true)
    }));

    return {
      iteration: it,
      objData,
      pfRate: _.sumBy(objData, d => d.dominanceMark) / _.size(objData)
    }
  }
), {
  iteration: 100,
  objData: Data.Proj.objData,
  pfRate: _.sumBy(Data.Proj.objData, d => d.dominanceMark) / _.size(Data.Proj.objData)
});

onMounted(() => {
  const svg = d3
    .select(".chart")
    .append("svg")
    .attr("width", 700)
    .attr("height", 700);
  const [resPF, resReg] = _.partition(Data.Proj.objData, p => p.dominanceMark === 1);

  // Defines
  svg.append('defs')
    .append('filter')
    .attr('id', 'blur');
  svg.select('filter')
    .append('feGaussianBlur')
    .attr('in', 'SourceGraphic')
    .attr('stdDeviation', 0.9);
  svg.select('defs')
    .append('linearGradient')
    .attr('id', 'colors')
    .attr('x1', '0%')
    // .attr('y1', '0%')
    .attr('x2', '100%')
    // .attr('y2', '100%')
    .attr("spreadMethod", "pad");
  svg.select('defs')
    .select('#colors')
    .append('stop')
    .attr('offset', '0%')
    .attr('stop-color', 'rgb(255, 255, 0)')
    .attr('stop-opacity', 1);
  svg.select('defs')
    .select('#colors')
    .append('stop')
    .attr('offset', '100%')
    .attr('stop-color', 'rgb(0, 176, 80)')
    .attr('stop-opacity', 1);

  // PF Reference
  svg.selectAll("pf-refs")
    .data(_.map(Data.Proj.pf, p => [p.x, p.y]))
    .enter()
    .append("circle")
    .attr("fill", d3.color("rgba(127, 127, 127, 0.1)"))
    .attr("filter", "url(#blur)")
    .attr("cx", d => xScale(d[0]))
    .attr("cy", d => yScale(d[1]))
    .attr("r", 6);

  // Result
  svg.selectAll("result-reg")
    .data(_.map(resReg, p => [p.x, p.y]))
    .enter()
    .append("circle")
    .attr('id', 'result-100')
    .attr("fill", color(0.5))
    .attr("cx", d => xScale(d[0]))
    .attr("cy", d => yScale(d[1]))
    .attr("stroke", 'gray')
    .attr("r", 6);
  svg.selectAll("result-pf")
    .data(_.map(resPF, p => [p.x, p.y]))
    .enter()
    .append("rect")
    .attr('id', 'result-100')
    .attr("fill", color(0.5))
    .attr("x", d => xScale(d[0]))
    .attr("y", d => yScale(d[1]))
    .attr("stroke", 'gray')
    .attr('width', 12)
    .attr('height', 12);

  const grp = d3
    .select(".graph")
    .append("svg")
    .attr("width", 700)
    .attr("height", 50);

  grp.append("path")
    .attr("fill", "rgb(196, 196, 196)")
    .attr("d", area(_.map(res, d => d.iteration)));
});

const update = (range) => {
  const oldRange = _.range(value.value[0], value.value[1] + 1, 10);
  value.value = range;
  nextTick(() => {
    const newRange = _.range(range[0], range[1] + 1, 10);
    const svg = d3
      .select(".chart")
      .select("svg");
    const linear = d3.scaleLinear()
      .domain(value.value)
      .range([0, 1]);

    _.forEach(_.without(oldRange, newRange), it => {
      svg.selectAll(`#result-${it}`)
        .remove();
    });
    _.forEach(_.intersection(oldRange, newRange), it => {
      svg.selectAll(`#result-${it}`)
        .attr("fill", color(linear(it)));
    });
    _.forEach(_.without(newRange, oldRange), it => {
      const [resPF, resReg] = _.partition(_.find(res, d => d.iteration === it).objData, p => p.dominanceMark === 1);
      svg.selectAll("result-reg")
        .data(_.map(resReg, p => [p.x, p.y]))
        .enter()
        .append("circle")
        .attr('id', `result-${it}`)
        .attr("fill", color(linear(it)))
        .attr("cx", d => xScale(d[0]))
        .attr("cy", d => yScale(d[1]))
        .attr("stroke", 'gray')
        .attr("r", 6);
      svg.selectAll("result-pf")
        .data(_.map(resPF, p => [p.x, p.y]))
        .enter()
        .append("rect")
        .attr('id', `result-${it}`)
        .attr("fill", color(linear(it)))
        .attr("x", d => xScale(d[0]))
        .attr("y", d => yScale(d[1]))
        .attr("stroke", 'gray')
        .attr('width', 12)
        .attr('height', 12);
    });

    const grp = d3
      .select(".graph")
      .select("svg");

    grp.selectAll("#current-range")
      .remove();
    grp.append("path")
      .attr("id", "current-range")
      .attr("fill", "url(#colors)")
      .attr("d", area(newRange));
  });
};
</script>

<style scoped>
.container {
  max-width: 50vw;
  max-height: 75vh;
}
</style>
