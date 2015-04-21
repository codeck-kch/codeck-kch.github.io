---
layout: default
title: 你好，世界
published: true
custom_js:
- mathjaxcn
- d3cn
---

我的第一篇文章,修改一下!

试试公式能不能用
$$a^2 + b^2 = c^2$$

再试试d3能不能用

<div id="viz">
<script>
var sampleSVG = d3.select("div#viz")
.append("svg")
.attr("width", 100)
.attr("height", 100);

sampleSVG.append("circle")
.style("stroke", "gray")
.style("fill", "white")
.attr("r", 40)
.attr("cx", 50)
.attr("cy", 50)
.on("mouseover", function(){d3.select(this).style("fill", "aliceblue");})
.on("mouseout", function(){d3.select(this).style("fill", "white");});
</script>
