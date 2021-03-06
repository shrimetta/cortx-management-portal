/*
* CORTX-CSM: CORTX Management web and CLI interface.
* Copyright (c) 2020 Seagate Technology LLC and/or its Affiliates
* This program is free software: you can redistribute it and/or modify
* it under the terms of the GNU Affero General Public License as published
* by the Free Software Foundation, either version 3 of the License, or
* (at your option) any later version.
* This program is distributed in the hope that it will be useful,
* but WITHOUT ANY WARRANTY; without even the implied warranty of
* MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
* GNU Affero General Public License for more details.
* You should have received a copy of the GNU Affero General Public License
* along with this program. If not, see <https://www.gnu.org/licenses/>.
* For any questions about this software or licensing,
* please email opensource@seagate.com or cortx-questions@seagate.com.
*/
<template>
  <div id="treeContainer"></div>
</template>
<script lang="ts">
import CortxTabs, { TabsInfo } from "../widgets/cortx-tabs.vue";
import { Component, Vue, Prop } from "vue-property-decorator";
import { Api } from "../../services/api";
import apiRegister from "../../services/api-register";
import * as d3 from "d3";
import * as moment from "moment";

@Component({
  name: "cortx-health-graphical"
})
export default class CortxHealthGraphical extends Vue {
  public clusterHealthData: any = {};

  public async mounted() {
    await this.getHealthData();
  }

  public async getHealthData() {
    const healthQueryParams = {
      depth: 0
    };
    this.$store.dispatch("systemConfig/showLoader", "Fetching health...");
    const res = await Api.getAll(apiRegister.health_cluster, healthQueryParams);
    this.clusterHealthData = res.data;
    this.$store.dispatch("systemConfig/hideLoader");
    this.plotTree();
  }

  public plotTree() {
    // Set the dimensions and margins of the diagram
    const margin = { top: 20, right: 90, bottom: 30, left: 90 };
    const width = 1000 - margin.left - margin.right;
    const height = 600 - margin.top - margin.bottom;
    // append the svg object to the body of the page
    // appends a 'group' element to 'svg'
    // moves the 'group' element to the top left margin
    const svg = d3
      .select("#treeContainer")
      .append("svg")
      .attr("width", width + margin.right + margin.left)
      .attr("height", height + margin.top + margin.bottom)
      .style("overflow", "auto")
      .append("g")
      .attr("transform", `translate( ${margin.left} , ${margin.top} )`);
    const duration = 750;
    // declares a tree layout and assigns the size
    const treemap = d3.tree().size([height, width]);
    // Assigns parent, children, height, depth
    const root: any = d3.hierarchy(this.clusterHealthData.data[0], function(
      d: any
    ) {
      return d.sub_resources;
    });
    root.x0 = 0;
    root.y0 = 0;
    // Collapse after the second level
    root.children.forEach(collapse);
    update(root);
    // Collapse the node and all it's children
    function collapse(d: any) {
      if (d.children && d.depth > 2) {
        d._children = d.children;
        d._children.forEach(collapse);
        d.children = null;
      } else if (d.children) {
        d.children.forEach(collapse);
      }
    }
    function update(source: any) {
      // Assigns the x and y position for the nodes
      const treeData = treemap(root);
      // Compute the new tree layout.
      const nodes = treeData.descendants(),
        links = treeData.descendants().slice(1);
      // Normalize for fixed-depth.
      nodes.forEach(function(d: any) {
        d.y = d.depth * 180;
      });
      // ****************** Nodes section ***************************
      // Update the nodes...
      let i = 0;
      const node: any = svg.selectAll("g.node").data(nodes, function(d: any) {
        return d.id || (d.id = ++i);
      });
      // Enter any new modes at the parent's previous position.
      const nodeEnter = node
        .enter()
        .append("g")
        .attr("class", "node")
        .attr("transform", function(d: any) {
          return `translate( ${source.y0} , ${source.x0} )`;
        });
      nodeEnter
        .append("rect")
        .attr("width", "140")
        .attr("height", "120")
        .attr("rx", 5)
        .style("fill", "#fff")
        .style("stroke", color)
        .style("stroke-width", 1);
      nodeEnter
        .append("rect")
        .attr("width", function(d: any) {
          let width = "70px";
          if (
            d.data.status == "online" ||
            d.data.status == "offline" ||
            d.data.status == "failed"
          ) {
            width = "70px";
          } else if (
            d.data.status == "unknown" ||
            d.data.status == "degraded"
          ) {
            width = "90px";
          }
          return width;
        })
        .attr("height", "25px")
        .attr("rx", 10)
        .attr("x", 10)
        .attr("y", 7)
        .style("fill", color);
      nodeEnter
        .append("text")
        .attr("dy", "24px")
        .attr("x", function(d: any) {
          let xValue = "68px";
          if (
            d.data.status == "online" ||
            d.data.status == "offline" ||
            d.data.status == "failed"
          ) {
            xValue = d.children || d._children ? "68px" : "24px";
          } else if (
            d.data.status == "unknown" ||
            d.data.status == "degraded"
          ) {
            xValue = d.children || d._children ? "86px" : "23px";
          }
          return xValue;
        })
        .attr("text-anchor", function(d: any) {
          return d.children || d._children ? "end" : "start";
        })
        .style("fill", "#fff")
        .style("text-transform", "uppercase")
        .style("font-size", "12px")
        .text(function(d: any) {
          return d.data.status;
        });
      nodeEnter
        .append("text")
        .attr("x", "10px")
        .attr("y", "60px")
        .attr("id", function(d: any) {
          return `nodeInfoTextID${d.id}${d.depth}`;
        })
        .style("visibility", "hidden")
        .append("tspan")
        .style("font-size", "10px")
        .text(function(d: any) {
          return `Name: ${d.data.resource}`;
        })
        .append("tspan")
        .style("font-size", "10px")
        .attr("x", "10px")
        .attr("dy", "15px")
        .attr("class", function(d: any) {
          if (d.data.id.length > 20) {
            return "id-text";
          }
        })
        .text(function(d: any) {
          return `ID: ${d.data.id}`;
        })
        .append("tspan")
        .attr("x", "10px")
        .attr("dy", "15px")
        .style("font-size", "10px")
        .text(function(d: any) {
          const newDate = new Date(d.data.last_updated_time * 1000);
          const time = moment.default(newDate).format("DD-MM-YYYY hh:mm A");
          return `Time: ${time}`;
        });
      nodeEnter
        .append("image")
        .attr("class", "node")
        .attr("xlink:href", require("@/assets/info-icon.svg/"))
        .attr("x", "115px")
        .attr("y", "10px")
        .attr("width", "20px")
        .attr("height", "20px")
        .style("cursor", "pointer")
        .on("click", function(d: any) {
          const nodeInfoText = document.getElementById(
              `nodeInfoTextID${d.id}${d.depth}`
            ),
            noneImage = document.getElementById(`noneImage${d.id}${d.depth}`),
            nodetextId = document.getElementById(`nodetextId${d.id}${d.depth}`),
            nodetextId1 = document.getElementById(
              `nodetextId1${d.id}${d.depth}`
            );
          if (window.getComputedStyle(nodeInfoText!).visibility === "hidden") {
            nodeInfoText!.setAttribute("style", "visibility: visible");
            noneImage!.setAttribute("style", "visibility: hidden");
            nodetextId!.setAttribute("style", "visibility: hidden");
            nodetextId1!.setAttribute("style", "visibility: hidden");
          } else {
            nodeInfoText!.setAttribute("style", "visibility: hidden");
            noneImage!.setAttribute("style", "visibility: visible");
            nodetextId!.setAttribute(
              "style",
              "font-size: 14px; font-weight: bold; text-transform: capitalize;"
            );
            nodetextId1!.setAttribute(
              "style",
              "visibility: visible; font-size: 12px"
            );
          }
        });
      nodeEnter
        .append("line")
        .attr("x1", "0px")
        .attr("x2", function(d: any) {
          if (d.data.sub_resources && d.data.sub_resources.length) {
            return 145;
          } else {
            return 140;
          }
        })
        .attr("y2", "40px")
        .attr("y1", "40px")
        .attr("stroke", color);
      nodeEnter
        .append("image")
        .attr("class", "node")
        .attr("id", function(d: any) {
          return `noneImage${d.id}${d.depth}`;
        })
        .attr("xlink:href", imageLink)
        .attr("x", "10px")
        .attr("y", "55px")
        .attr("width", "30px")
        .attr("height", "30px");
      nodeEnter
        .append("text")
        .attr("id", function(d: any) {
          return `nodetextId${d.id}${d.depth}`;
        })
        .attr("dy", "70px")
        .attr("x", function(d: any) {
          return 50;
        })
        .style("font-size", "14px")
        .text(function(d: any) {
          return d.data.resource;
        })
        .style("font-weight", "bold")
        .style("text-transform", "capitalize");
      nodeEnter
        .append("text")
        .attr("id", function(d: any) {
          return `nodetextId1${d.id}${d.depth}`;
        })
        .attr("dy", "90px")
        .attr("x", function(d: any) {
          return 50;
        })
        .style("font-size", "12px")
        .text(function(d: any) {
          if (d.data.id.length > 5) {
            return `${d.data.id.substring(0, 5)}...`;
          } else {
            return d.data.id;
          }
        });
      nodeEnter
        .append("circle")
        .attr("class", "node")
        .attr("r", function(d: any) {
          if (d.data.sub_resources && d.data.sub_resources.length) {
            return "10";
          } else {
            return "0";
          }
        })
        .attr("cx", 155)
        .attr("cy", 40)
        .style("fill", "none")
        .style("stroke", "#6EBE49")
        .style("stroke-width", 1)
        .style("cursor", "pointer")
        .on("click", click);
      nodeEnter
        .append("text")
        .attr("class", "node-text")
        .attr("dy", "45px")
        .attr("x", function(d: any) {
          return d._children ? "150" : "152.5";
        })
        .text(function(d: any) {
          if (d.data.sub_resources && d.data.sub_resources.length) {
            return d._children ? "+" : "-";
          } else {
            return "";
          }
        })
        .style("cursor", "pointer")
        .on("click", click);
      // UPDATE
      const nodeUpdate: any = nodeEnter.merge(node);
      // Transition to the proper position for the node
      nodeUpdate
        .transition()
        .duration(duration)
        .attr("transform", function(d: any) {
          return `translate(${d.y} , ${d.x})`;
        });
      // Remove any exiting nodes
      const nodeExit = node
        .exit()
        .transition()
        .duration(duration)
        .attr("transform", function(d: any) {
          return `translate(${source.y} , ${source.x})`;
        })
        .remove();
      // On exit reduce the node circles size to 0
      nodeExit
        .select("rect")
        .attr("width", "140")
        .attr("height", "100")
        .attr("stroke-width", 1);

      node
        .select("text.node-text")
        .attr("x", function(d: any) {
          return d._children ? "150" : "152.5";
        })
        .text(function(d: any) {
          if (d.data.sub_resources && d.data.sub_resources.length) {
            return d._children ? "+" : "-";
          } else {
            return "";
          }
        });
      // ****************** links section ***************************
      // Update the links...
      const link: any = svg
        .selectAll("path.link")
        .data(links, function(d: any) {
          return d.id;
        });
      // Enter any new links at the parent's previous position.
      const linkEnter = link
        .enter()
        .insert("path", "g")
        .attr("class", "link")
        .attr("d", function(d: any) {
          const o = { x: source.x0, y: source.y0 };
          return diagonal(o, o);
        })
        .style("fill", "none")
        .style("stroke", "#0832A0")
        .style("stroke-width", "1px")
        .style("stroke-dasharray", "2 2");
      // UPDATE
      const linkUpdate: any = linkEnter.merge(link);
      // Transition back to the parent element position
      linkUpdate
        .transition()
        .duration(duration)
        .attr("d", function(d: any) {
          return diagonal(d, d.parent);
        });
      // Remove any exiting links
      const linkExit = link
        .exit()
        .transition()
        .duration(duration)
        .attr("d", function(d: any) {
          const o = { x: source.x, y: source.y };
          return diagonal(o, o);
        })
        .remove();

      // Store the old positions for transition.
      nodes.forEach(function(d: any) {
        d.x0 = d.x;
        d.y0 = d.y;
      });
      // Creates a curved (diagonal) path from parent to the child nodes
      function diagonal(s: any, d: any) {
        const path = `M ${s.y} ${s.x + 40}
                L ${d.y + 165} ${d.x + 40}`;
        return path;
      }
      // wrap text if so long
      d3.selectAll(".id-text").call(wrap);
      function wrap(text: any) {
        if (text.text().length > 20) {
          const idLine = text.text().slice(0, 20);
          const idLineNew = text.text().slice(20, text.text().indexOf("Time"));
          const time = text
            .text()
            .slice(text.text().indexOf("Time"), text.text().length);
          text
            .text(idLine)
            .append("tspan")
            .attr("x", text.attr("x"))
            .attr("dy", text.attr("y") + 15)
            .text(idLineNew)
            .append("tspan")
            .attr("x", text.attr("x"))
            .attr("dy", text.attr("y") + 15)
            .text(time);
        }
      }
      // Toggle children on click.
      function click(d: any) {
        if (d.children) {
          d._children = d.children;
          d.children = null;
        } else {
          d.children = d._children;
          d._children = null;
        }
        update(d);
      }
      // Return color value based on status
      function color(d: any) {
        let color = "";
        switch (d.data.status) {
          case "online":
            color = "#6ebe49";
            break;
          case "offline":
            color = "#dc1f2e";
            break;
          case "failed":
            color = "#dc1f2e";
            break;
          case "degraded":
            color = "#f7a528";
            break;
          case "unknown":
            color = "#9e9e9e";
            break;
        }
        return color;
      }
      // return image link based on status
      function imageLink(d: any) {
        let imageLink = "";
        switch (d.data.status) {
          case "online":
            imageLink = require("@/assets/green-icon.svg/");
            break;
          case "offline":
            imageLink = require("@/assets/red-icon.svg/");
            break;
          case "failed":
            imageLink = require("@/assets/red-icon.svg/");
            break;
          case "degraded":
            imageLink = require("@/assets/yellow-icon.svg/");
            break;
          case "unknown":
            imageLink = require("@/assets/grey-icon.svg/");
            break;
        }
        return imageLink;
      }
    }
  }
}
</script>
<style lang="scss" scoped>
#treeContainer {
  width: 100%;
  overflow: auto;
}
.link {
  fill: none;
  stroke: #ccc;
  stroke-width: 2px;
}
</style>
