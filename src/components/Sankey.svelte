<script>
	import { onMount } from 'svelte';
	import * as d3 from 'd3';

	let svgElement;
	let data = null;

	function createSankey(data) {
		if (!data || !svgElement) return;

		d3.select(svgElement).selectAll('*').remove();

		const width = 800;
		const height = 1200;
		const margin = { top: 40, right: 200, bottom: 40, left: 200 };
		const innerWidth = width - margin.left - margin.right;
		const innerHeight = height - margin.top - margin.bottom;

		const svg = d3
			.select(svgElement)
			.attr('width', width)
			.attr('height', height)
			.style('background', 'white');

		const g = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);

		// Procesar datos y agrupar países con valores < 11 en "Others"
		const processedNodes = new Map();
		const processedLinks = [];

		// Inicializar todos los nodos
		data.nodes.forEach((node) => {
			processedNodes.set(node.id, {
				...node,
				sourceLinks: [],
				targetLinks: [],
				outValue: 0,
				inValue: 0,
				totalValue: 0
			});
		});

		// Calcular valores totales por nodo
		data.links.forEach((link) => {
			const sourceNode = processedNodes.get(link.source);
			const targetNode = processedNodes.get(link.target);

			if (sourceNode && targetNode) {
				sourceNode.outValue += link.value;
				targetNode.inValue += link.value;
			}
		});

		// Identificar países que van a "Others"
		const othersSourceCountries = new Set();
		const othersTargetCountries = new Set();

		processedNodes.forEach((node, id) => {
			if (node.outValue > 0 && node.outValue < 11) {
				othersSourceCountries.add(id);
			}
			if (node.inValue > 0 && node.inValue < 11) {
				othersTargetCountries.add(id);
			}
		});

		// Crear nodos Others
		if (othersSourceCountries.size > 0) {
			processedNodes.set('Others_source', {
				id: 'Others_source',
				name: 'Others',
				sourceLinks: [],
				targetLinks: [],
				outValue: 0,
				inValue: 0,
				totalValue: 0
			});
		}

		if (othersTargetCountries.size > 0) {
			processedNodes.set('Others_target', {
				id: 'Others_target',
				name: 'Others',
				sourceLinks: [],
				targetLinks: [],
				outValue: 0,
				inValue: 0,
				totalValue: 0
			});
		}

		// Procesar enlaces y agrupar en Others
		data.links.forEach((link) => {
			let sourceId = link.source;
			let targetId = link.target;

			// Si el source va a Others
			if (othersSourceCountries.has(link.source)) {
				sourceId = 'Others_source';
			}

			// Si el target va a Others
			if (othersTargetCountries.has(link.target)) {
				targetId = 'Others_target';
			}

			// Buscar si ya existe un enlace combinado
			const existingLinkIndex = processedLinks.findIndex(
				(l) => l.source === sourceId && l.target === targetId
			);

			if (existingLinkIndex >= 0) {
				processedLinks[existingLinkIndex].value += link.value;
			} else {
				processedLinks.push({
					source: sourceId,
					target: targetId,
					value: link.value
				});
			}
		});

		// Recalcular valores después del agrupamiento
		processedNodes.forEach((node) => {
			node.outValue = 0;
			node.inValue = 0;
		});

		processedLinks.forEach((link) => {
			const sourceNode = processedNodes.get(link.source);
			const targetNode = processedNodes.get(link.target);

			if (sourceNode && targetNode) {
				sourceNode.sourceLinks.push(link);
				targetNode.targetLinks.push(link);
				sourceNode.outValue += link.value;
				targetNode.inValue += link.value;
			}
		});

		// Filtrar nodos que no tienen flujos
		const activeNodes = Array.from(processedNodes.values()).filter(
			(node) => node.outValue > 0 || node.inValue > 0
		);

		activeNodes.forEach((node) => {
			node.sourceLinks.sort((a, b) => b.value - a.value);
			node.targetLinks.sort((a, b) => b.value - a.value);
			node.totalValue = Math.max(node.outValue, node.inValue, 1);
		});

		// Crear nodos source y target separados
		const sourceNodes = activeNodes
			.filter((n) => n.outValue > 0)
			.map((n) => ({
				...n,
				id: n.id + '_source_display',
				originalId: n.id,
				type: 'source',
				value: Math.max(n.outValue, 1),
				displayValue: n.outValue
			}));

		const targetNodes = activeNodes
			.filter((n) => n.inValue > 0)
			.map((n) => ({
				...n,
				id: n.id + '_target_display',
				originalId: n.id,
				type: 'target',
				value: Math.max(n.inValue, 1),
				displayValue: n.inValue
			}));

		console.log(`Nodos activos fuente: ${sourceNodes.length}`);
		console.log(`Nodos activos destino: ${targetNodes.length}`);
		console.log(`Enlaces procesados: ${processedLinks.length}`);

		// Ordenar con Others al final
		sourceNodes.sort((a, b) => {
			if (a.name === 'Others') return 1;
			if (b.name === 'Others') return -1;
			return b.value - a.value;
		});

		targetNodes.sort((a, b) => {
			if (a.name === 'Others') return 1;
			if (b.name === 'Others') return -1;
			return b.value - a.value;
		});

		const nodeWidth = 15;
		const nodePadding = 5;
		const sourceTotal = d3.sum(sourceNodes, (d) => d.value);
		const targetTotal = d3.sum(targetNodes, (d) => d.value);

		const sourceScale = (innerHeight - (sourceNodes.length - 1) * nodePadding) / sourceTotal;
		const targetScale = (innerHeight - (targetNodes.length - 1) * nodePadding) / targetTotal;

		// Posicionar nodos source
		let y = 0;
		sourceNodes.forEach((node) => {
			node.x0 = 0;
			node.x1 = nodeWidth;
			node.y0 = y;
			node.y1 = y + node.value * sourceScale;
			node.linkY = node.y0;
			y = node.y1 + nodePadding;
		});

		// Posicionar nodos target
		y = 0;
		targetNodes.forEach((node) => {
			node.x0 = innerWidth - nodeWidth;
			node.x1 = innerWidth;
			node.y0 = y;
			node.y1 = y + node.value * targetScale;
			node.linkY = node.y0;
			y = node.y1 + nodePadding;
		});

		const allNodes = [...sourceNodes, ...targetNodes];

		// Crear escala de colores con Others al final en gris
		const allCountries = [...new Set(activeNodes.map((d) => d.name))].sort();
		const othersIndex = allCountries.indexOf('Others');
		if (othersIndex > -1) {
			allCountries.splice(othersIndex, 1);
			allCountries.push('Others');
		}

		const colors = [
			'#1f77b4',
			'#ff7f0e',
			'#2ca02c',
			'#d62728',
			'#9467bd',
			'#8c564b',
			'#e377c2',
			'#7f7f7f',
			'#bcbd22',
			'#17becf',
			'#aec7e8',
			'#ffbb78',
			'#98df8a',
			'#ff9896',
			'#c5b0d5',
			'#c49c94',
			'#f7b6d3',
			'#c7c7c7',
			'#dbdb8d',
			'#9edae5',
			'#ff6b6b',
			'#4ecdc4',
			'#45b7d1',
			'#96ceb4',
			'#feca57',
			'#ff9ff3',
			'#54a0ff',
			'#5f27cd',
			'#00d2d3',
			'#ff9f43',
			'#10ac84',
			'#ee5a24',
			'#0984e3',
			'#a29bfe',
			'#fd79a8',
			'#fdcb6e',
			'#6c5ce7',
			'#fab1a0',
			'#00b894',
			'#e84393',
			'#0984e3',
			'#74b9ff'
		];

		const colorScale = d3
			.scaleOrdinal()
			.domain(allCountries)
			.range([...colors.slice(0, allCountries.length - 1), '#888888']); // Others en gris

		console.log(`Países únicos: ${allCountries.length}`);
		allCountries.forEach((country) => console.log(`- ${country}`));

		function createLinkPath(link) {
			const sourceNode = sourceNodes.find((n) => n.originalId === link.source);
			const targetNode = targetNodes.find((n) => n.originalId === link.target);

			if (!sourceNode || !targetNode) {
				console.log(`No se encontró nodo para enlace: ${link.source} -> ${link.target}`);
				return '';
			}

			// Calcular altura del enlace proporcional al valor y al tamaño del nodo
			const sourceNodeHeight = sourceNode.y1 - sourceNode.y0;
			const targetNodeHeight = targetNode.y1 - targetNode.y0;

			// La altura del enlace debe ser proporcional al valor dentro del nodo
			const sourceLinkHeight = (link.value / sourceNode.displayValue) * sourceNodeHeight;
			const targetLinkHeight = (link.value / targetNode.displayValue) * targetNodeHeight;

			const linkHeight = Math.min(sourceLinkHeight, targetLinkHeight);

			// Verificar que no se pase del límite del nodo
			const sourceY0 = sourceNode.linkY;
			const sourceY1 = Math.min(sourceY0 + linkHeight, sourceNode.y1);
			const actualSourceHeight = sourceY1 - sourceY0;

			const targetY0 = targetNode.linkY;
			const targetY1 = Math.min(targetY0 + linkHeight, targetNode.y1);
			const actualTargetHeight = targetY1 - targetY0;

			// Actualizar la posición para el siguiente enlace
			sourceNode.linkY = sourceY1;
			targetNode.linkY = targetY1;

			const x0 = sourceNode.x1;
			const x1 = targetNode.x0;
			const y0 = sourceY0 + actualSourceHeight / 2;
			const y1 = targetY0 + actualTargetHeight / 2;

			const curvature = 0.6;
			const xi = d3.interpolateNumber(x0, x1);
			const x2 = xi(curvature);
			const x3 = xi(1 - curvature);

			link.y0 = y0;
			link.y1 = y1;
			link.thickness = Math.max(actualSourceHeight, actualTargetHeight);

			return `M${x0},${y0}C${x2},${y0} ${x3},${y1} ${x1},${y1}`;
		}

		const validLinks = processedLinks.filter((link) => {
			const sourceExists = sourceNodes.some((n) => n.originalId === link.source);
			const targetExists = targetNodes.some((n) => n.originalId === link.target);
			return sourceExists && targetExists;
		});

		console.log(`Enlaces válidos: ${validLinks.length} de ${processedLinks.length}`);

		// Dibujar enlaces
		const linkGroup = g.append('g').attr('class', 'links');
		const linkPaths = linkGroup
			.selectAll('path')
			.data(validLinks)
			.join('path')
			.attr('d', createLinkPath)
			.attr('stroke', (d) => {
				const sourceNode = processedNodes.get(d.source);
				return sourceNode ? colorScale(sourceNode.name) : '#999';
			})
			.attr('stroke-width', (d) => d.thickness || Math.max(1, Math.sqrt(d.value) * 2))
			.attr('stroke-opacity', 0.7)
			.attr('fill', 'none');

		// Dibujar nodos
		const nodeGroup = g.append('g').attr('class', 'nodes');
		const nodeRects = nodeGroup
			.selectAll('rect')
			.data(allNodes)
			.join('rect')
			.attr('x', (d) => d.x0)
			.attr('y', (d) => d.y0)
			.attr('height', (d) => Math.max(2, d.y1 - d.y0))
			.attr('width', (d) => d.x1 - d.x0)
			.attr('fill', (d) => colorScale(d.name))
			.attr('stroke', '#fff')
			.attr('stroke-width', 0.5);

		// Dibujar etiquetas
		const labelGroup = g.append('g').attr('class', 'labels');
		const labels = labelGroup
			.selectAll('text')
			.data(allNodes)
			.join('text')
			.attr('x', (d) => (d.type === 'source' ? d.x0 - 8 : d.x1 + 8))
			.attr('y', (d) => (d.y1 + d.y0) / 2)
			.attr('dy', '0.35em')
			.attr('text-anchor', (d) => (d.type === 'source' ? 'end' : 'start'))
			.style('font-family', 'Arial, sans-serif')
			.style('font-size', '11px')
			.style('fill', '#333')
			.style('font-weight', '500')
			.text((d) => `${d.name} (${d.displayValue})`);

		// Tooltip
		const tooltip = d3
			.select('body')
			.append('div')
			.attr('class', 'sankey-tooltip')
			.style('position', 'absolute')
			.style('visibility', 'hidden')
			.style('background', 'rgba(0, 0, 0, 0.9)')
			.style('color', 'white')
			.style('padding', '8px 12px')
			.style('border-radius', '4px')
			.style('font-size', '12px')
			.style('pointer-events', 'none')
			.style('z-index', '1000');

		linkPaths
			.on('mouseover', function (event, d) {
				d3.select(this).attr('stroke-opacity', 0.9);
				const sourceNode = processedNodes.get(d.source);
				const targetNode = processedNodes.get(d.target);
				tooltip
					.style('visibility', 'visible')
					.html(`${sourceNode.name} → ${targetNode.name}<br/>Valor: ${d.value}`);
			})
			.on('mousemove', function (event) {
				tooltip.style('top', event.pageY - 10 + 'px').style('left', event.pageX + 10 + 'px');
			})
			.on('mouseout', function () {
				d3.select(this).attr('stroke-opacity', 0.6);
				tooltip.style('visibility', 'hidden');
			});

		nodeRects
			.on('mouseover', function (event, d) {
				tooltip
					.style('visibility', 'visible')
					.html(`${d.name}<br/>Tipo: ${d.type}<br/>Valor: ${d.displayValue}`);
			})
			.on('mousemove', function (event) {
				tooltip.style('top', event.pageY - 10 + 'px').style('left', event.pageX + 10 + 'px');
			})
			.on('mouseout', function () {
				tooltip.style('visibility', 'hidden');
			});
	}

	onMount(async () => {
		try {
			const response = await fetch('./sankey_data.json');
			data = await response.json();
			createSankey(data);
		} catch (error) {
			console.error('Error cargando los datos:', error);
		}
	});
</script>

<div class="sankey-container">
	<svg bind:this={svgElement} />
</div>

<style>
	.sankey-container {
		width: 100%;
		background: white;
		padding: 20px;
		font-family: Arial, sans-serif;
	}

	svg {
		display: block;
		margin: 0 auto;
		border: 1px solid #ddd;
	}
</style>
