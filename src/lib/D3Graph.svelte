<script lang="ts">
  // @ts-nocheck
  import { onMount } from 'svelte'
  import * as d3 from 'd3'

  let containerEl
  let svgEl
  let tooltipEl

  const H = 560
  const PAD = { top: 20, right: 160, bottom: 30, left: 160 }
  const NODE_W = 14
  const LINK_OPACITY = 0.35

  let nodes = []
  let mineralCountry = []
  let countryProduct = []
  let allLinks = []

  const getColors = () => {
    const isDark = matchMedia('(prefers-color-scheme: dark)').matches
    return {
      isDark,
      mineral: isDark ? '#9F9AE8' : '#534AB7',
      country: isDark ? '#5DCAA5' : '#0F6E56',
      product: isDark ? '#F0997B' : '#993C1D'
    }
  }

  function layoutNodes(layoutNodesRef, links, w) {
    const col = [PAD.left, w / 2, w - PAD.right]
    const cols = [[], [], []]
    layoutNodesRef.forEach((n) => cols[n.col].push(n))

    const innerH = H - PAD.top - PAD.bottom
    const GAP = 10

    cols.forEach((group, ci) => {
      group.forEach((n) => {
        let v = 0
        if (ci === 0) v = links.filter((l) => l.s === n.id).reduce((a, l) => a + l.v, 0)
        else if (ci === 1) {
          v =
            links
              .filter((l) => l.t === n.id || l.s === n.id)
              .reduce((a, l) => {
                if (l.t === n.id) return a + l.v
                if (l.s === n.id) return a + l.v
                return a
              }, 0) / 2
        } else {
          v = links.filter((l) => l.t === n.id).reduce((a, l) => a + l.v, 0)
        }
        n._val = v
      })

      const totalVal = group.reduce((sum, n) => sum + n._val, 0)
      const totalGap = GAP * (group.length - 1)
      const available = innerH - totalGap

      let y = PAD.top
      group.forEach((n) => {
        const h = (n._val / totalVal) * available
        n._y = y
        n._h = Math.max(h, 4)
        y += n._h + GAP
      })
    })

    return col
  }

  function buildSankey() {
    if (!nodes.length || !mineralCountry.length || !countryProduct.length) return
    const colors = getColors()
    const W = Math.max(760, Math.min(960, containerEl.clientWidth || window.innerWidth - 32))
    const col = [PAD.left, W / 2, W - PAD.right]

    const localNodes = nodes.map((n) => ({ ...n }))
    const localLinks = [...allLinks]
    const nodeMap = Object.fromEntries(localNodes.map((n) => [n.id, n]))

    layoutNodes(localNodes, localLinks, W)
    localNodes.forEach((n) => {
      n._outY = n._y
      n._inY = n._y
    })

    layoutNodes(localNodes, localLinks, W)
    localNodes.forEach((n) => {
      n._outY = n._y
      n._inY = n._y
    })

    const svg = d3.select(svgEl)
    svg.selectAll('*').remove()
    svg.attr('viewBox', `0 0 ${W} ${H}`).attr('width', '100%').style('overflow', 'visible')

    const sumBySource = (links) => {
      const totals = {}
      links.forEach((l) => {
        totals[l.s] = (totals[l.s] ?? 0) + l.v
      })
      return totals
    }

    const sumByTarget = (links) => {
      const totals = {}
      links.forEach((l) => {
        totals[l.t] = (totals[l.t] ?? 0) + l.v
      })
      return totals
    }

    function drawLinks(links, sourceCol, sourceTotals, targetTotals) {
      const bySource = d3.group(links, (l) => l.s)
      bySource.forEach((group, sid) => {
        const sourceNode = nodeMap[sid]
        const total = sourceTotals[sid] ?? group.reduce((sum, l) => sum + l.v, 0)

        group.forEach((link) => {
          const targetNode = nodeMap[link.t]
          const frac = link.v / (total || 1)
          const sh = sourceNode._h * frac
          const targetTotal = targetTotals[link.t] ?? targetNode._val ?? 1
          const th = targetNode._h * (link.v / targetTotal)

          const sy0 = sourceNode._outY
          const sy1 = sy0 + sh
          sourceNode._outY = sy1

          const ty0 = targetNode._inY
          const ty1 = ty0 + th
          targetNode._inY = ty1

          const x0 = col[sourceCol] + NODE_W
          const x1 = col[sourceCol + 1]
          const mx = (x0 + x1) / 2
          const pathD = `M${x0},${sy0} C${mx},${sy0} ${mx},${ty0} ${x1},${ty0} L${x1},${ty1} C${mx},${ty1} ${mx},${sy1} ${x0},${sy1} Z`
          const color = sourceCol === 0 ? colors.mineral : colors.country

          svg
            .append('path')
            .attr('d', pathD)
            .attr('fill', color)
            .attr('opacity', LINK_OPACITY)
            .on('mouseover', (event) => {
              const side = sourceCol === 0 ? 'refining/processing' : 'manufacturing'
              tooltipEl.innerHTML = `<strong>${link.s} → ${link.t}</strong><br>${link.v}% share of global ${side}`
              tooltipEl.style.opacity = '1'
              d3.select(event.currentTarget).attr('opacity', 0.7)
            })
            .on('mousemove', (event) => {
              tooltipEl.style.left = `${event.pageX + 12}px`
              tooltipEl.style.top = `${event.pageY - 28}px`
            })
            .on('mouseout', (event) => {
              tooltipEl.style.opacity = '0'
              d3.select(event.currentTarget).attr('opacity', LINK_OPACITY)
            })
        })
      })
    }

    const mineralSourceTotals = sumBySource(mineralCountry)
    const countryIncomingTotals = sumByTarget(mineralCountry)
    const countryOutgoingTotals = sumBySource(countryProduct)
    const productIncomingTotals = sumByTarget(countryProduct)

    drawLinks(mineralCountry, 0, mineralSourceTotals, countryIncomingTotals)
    localNodes.forEach((n) => {
      n._inY = n._y
    })
    drawLinks(countryProduct, 1, countryOutgoingTotals, productIncomingTotals)

    const colColors = [colors.mineral, colors.country, colors.product]
    localNodes.forEach((n) => {
      const x = col[n.col]
      const color = colColors[n.col]

      svg.append('rect').attr('x', x).attr('y', n._y).attr('width', NODE_W).attr('height', n._h).attr('fill', color).attr('rx', 2)

      const isRight = n.col === 2
      const tx = isRight ? x + NODE_W + 8 : x - 8
      const anchor = isRight ? 'start' : 'end'

      svg
        .append('text')
        .attr('x', tx)
        .attr('y', n._y + n._h / 2)
        .attr('dy', '0.35em')
        .attr('text-anchor', anchor)
        .attr('font-size', 13)
        .attr('font-family', 'sans-serif')
        .attr('fill', colors.isDark ? '#e5e4de' : '#2C2C2A')
        .text(n.label)
    })

    const headers = ['Minerals', 'Countries', 'Products']
    ;[0, 1, 2].forEach((ci) => {
      svg
        .append('text')
        .attr('x', col[ci] + NODE_W / 2)
        .attr('y', PAD.top - 8)
        .attr('text-anchor', 'middle')
        .attr('font-size', 12)
        .attr('font-family', 'sans-serif')
        .attr('fill', colors.isDark ? '#888780' : '#5F5E5A')
        .attr('font-weight', 500)
        .text(headers[ci])
    })

    const legendData = [
      { label: 'Mineral flow', color: colors.mineral },
      { label: 'Manufacturing flow', color: colors.country }
    ]
    const lx = W - PAD.right + 20
    legendData.forEach((d, i) => {
      svg.append('rect').attr('x', lx).attr('y', H - 60 + i * 22).attr('width', 12).attr('height', 12).attr('fill', d.color).attr('rx', 2)
      svg
        .append('text')
        .attr('x', lx + 18)
        .attr('y', H - 54 + i * 22)
        .attr('font-size', 12)
        .attr('font-family', 'sans-serif')
        .attr('fill', colors.isDark ? '#B4B2A9' : '#5F5E5A')
        .text(d.label)
    })
  }

  onMount(() => {
    let mounted = true
    const redraw = () => buildSankey()

    const loadData = async () => {
      const response = await fetch('/sankey-data.json')
      const data = await response.json()
      if (!mounted) return

      nodes = data.nodes ?? []
      mineralCountry = data.mineralCountry ?? []
      countryProduct = data.countryProduct ?? []
      allLinks = [...mineralCountry, ...countryProduct]
      buildSankey()
    }

    loadData()
    window.addEventListener('resize', redraw)
    return () => {
      mounted = false
      window.removeEventListener('resize', redraw)
    }
  })
</script>

<section class="graph-wrap">
  <div class="graph-title">Critical minerals to clean energy supply chain</div>
  <div class="sankey-container" bind:this={containerEl}>
    <svg bind:this={svgEl} aria-label="Critical minerals Sankey graph"></svg>
  </div>
  <div id="tooltip" bind:this={tooltipEl}></div>
</section>

<style>
  .graph-wrap {
    position: relative;
    padding: 16px;
    border-radius: 12px;
  }

  .graph-title {
    font-family: sans-serif;
    font-size: 20px;
    line-height: 1.3;
    margin-bottom: 12px;
    color: #2c2c2a;
  }

  .sankey-container {
    width: 100%;
    overflow-x: auto;
  }

  #tooltip {
    position: fixed;
    background: #fff;
    border: 0.5px solid #ccc;
    border-radius: 8px;
    padding: 8px 12px;
    font-size: 13px;
    pointer-events: none;
    opacity: 0;
    transition: opacity 0.15s;
    max-width: 220px;
    z-index: 10;
    color: #2c2c2a;
    font-family: sans-serif;
  }

  @media (prefers-color-scheme: dark) {
    .graph-title {
      color: #e5e4de;
    }

    #tooltip {
      background: #2c2c2a;
      border-color: #555;
      color: #e5e4de;
    }
  }
</style>
