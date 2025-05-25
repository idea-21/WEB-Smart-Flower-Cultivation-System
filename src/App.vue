<script setup>
	import {
		ref,
		onMounted,
		onUnmounted,
		watch,
		nextTick
	} from 'vue';
	import axios from 'axios';

	import * as echarts from 'echarts/core';
	import {
		TitleComponent,
		TooltipComponent,
		GridComponent,
		LegendComponent
	} from 'echarts/components';
	import {
		LineChart
	} from 'echarts/charts';
	import {
		UniversalTransition
	} from 'echarts/features';
	import {
		CanvasRenderer
	} from 'echarts/renderers';

	echarts.use([
		TitleComponent,
		TooltipComponent,
		GridComponent,
		LegendComponent,
		LineChart,
		CanvasRenderer,
		UniversalTransition
	]);


	// --- Configuration ---
	const cloudFunctionUrl =
	'https://fc-mp-f7ed7a70-910e-4dc2-b5b8-6f6f2f326d72.next.bspapp.com/MyFun'; // Your cloud function URL

	// --- Reactive State ---
	const temperature = ref('--');
	const humidity = ref('--');
	const soilValue = ref('--');
	const rainValue = ref('--');
	const Co = ref('--');
	const historicalData = ref([]);
	const isLoadingCurrent = ref(false);
	const isLoadingHistory = ref(false);
	const error = ref(null);
	let intervalId = null;

	// --- Chart Specific Refs ---
	const chartContainer = ref(null); // Ref for the chart DOM element
	let chartInstance = null; // To hold the ECharts instance

	// --- Data Fetching Functions ---
	const fetchLatestData = async () => {
		isLoadingCurrent.value = true;
		error.value = null;
		try {
			const response = await axios.post(cloudFunctionUrl, {
				action: 'get',
				limit: 1
			});
			console.log('API Response (Latest):', response.data);
			if (response.data && response.data.success && response.data.data.length > 0) {
				const latestRecord = response.data.data[0];
				temperature.value = latestRecord.temperature;
				humidity.value = latestRecord.humidity;
				soilValue.value = latestRecord.soilValue !== undefined ? latestRecord.soilValue : '--';
				rainValue.value = latestRecord.rainValue !== undefined ? latestRecord.rainValue : '--';
				Co.value = latestRecord.Co;
			} else {
				console.warn('Could not fetch latest data or no data available:', response.data?.message);
			}
		} catch (err) {
			console.error('Error fetching latest data:', err);
			if (err.message.includes('Network Error')) {
				error.value = '网络错误，无法连接到服务器。';
			} else if (err.response && err.response.status === 403) {
				error.value = '跨域请求被阻止 (CORS)，请检查服务器配置。';
			} else {
				error.value = '加载最新数据时出错。';
			}
			temperature.value = '错误';
			humidity.value = '错误';
			soilValue.value = '错误';
			rainValue.value = '错误';
			Co.value = '错误';
		} finally {
			isLoadingCurrent.value = false;
		}
	};

	const fetchHistoricalData = async () => {
		isLoadingHistory.value = true;
		if (error.value && error.value.includes('历史数据')) {
			error.value = null;
		}
		try {
			const response = await axios.post(cloudFunctionUrl, {
				action: 'get',
				limit: 7
			});
			console.log('API Response (History):', response.data);
			if (response.data && response.data.success && Array.isArray(response.data.data)) {
				historicalData.value = response.data.data.reverse();
			} else {
				const message = response.data?.message || '无法获取历史数据格式';
				throw new Error(message);
			}
		} catch (err) {
			console.error('Error fetching historical data:', err);
			if (!error.value) {
				error.value = `加载历史数据时出错: ${err.message}`;
			}
			historicalData.value = [];
		} finally {
			isLoadingHistory.value = false;
		}
	};

	// --- Formatting Functions ---
	const formatTimestamp = (timestamp) => {
		if (!timestamp) return 'N/A';
		const ts = (typeof timestamp === 'object' && timestamp !== null && typeof timestamp.$date === 'number') ?
			timestamp.$date :
			Number(timestamp);

		if (isNaN(ts)) return 'Invalid Date';

		const date = new Date(ts);
		return date.toLocaleString('zh-CN', {
			year: 'numeric',
			month: '2-digit',
			day: '2-digit',
			hour: '2-digit',
			minute: '2-digit',
			second: '2-digit',
			hour12: false
		});
	};

	const formatChartTimestamp = (timestamp) => {
		if (!timestamp) return '';
		const ts = (typeof timestamp === 'object' && timestamp !== null && typeof timestamp.$date === 'number') ?
			timestamp.$date :
			Number(timestamp);
		if (isNaN(ts)) return '';
		const date = new Date(ts);
		const m = (date.getMinutes() < 10 ? '0' + date.getMinutes() : date.getMinutes()) + ':';
		const s = (date.getSeconds() < 10 ? '0' + date.getSeconds() : date.getSeconds());
		return m + s;
	};


	// --- Chart Logic ---

	// Function to generate ECharts options based on historicalData
	const getChartOption = (data) => {
		if (!data || data.length === 0) {
			// Return a default state or message when no data
			return {
				title: {
					text: '暂无图表数据',
					left: 'center',
					top: 'center',
					textStyle: {
						color: '#888'
					}
				},
				xAxis: {
					show: false
				},
				yAxis: {
					show: false
				},
				series: []
			};
		}

		const categories = data.map(item => formatChartTimestamp(item.timestamp));
		const tempData = data.map(item => Number(item.temperature) || 0);
		const humData = data.map(item => Number(item.humidity) || 0);
		const soilData = data.map(item => Number(item.soilValue) || 0);
		const rainData = data.map(item => Number(item.rainValue) || 0);
		const Co = data.map(item => Number(item.Co) || 0);

		return {
			tooltip: {
				trigger: 'axis',
				axisPointer: {
					type: 'cross',
					label: {
						backgroundColor: '#6a7985'
					}
				},
				formatter: function(params) {
					let res = params[0].name + '<br/>';
					params.forEach(item => {
						res +=
							`${item.marker} ${item.seriesName}: ${item.value || '-'} ${getUnit(item.seriesName)}<br/>`;
					});
					return res;
				}
			},
			legend: {
				data: ['温度(℃)', '湿度(%)', '土壤湿度(%)', '雨量(%)', "CO2/10(ppm)"],
				bottom: 10
			},
			grid: {
				left: '3%',
				right: '4%',
				bottom: '10%',
				top: '10%',
				containLabel: true
			},
			xAxis: {
				type: 'category',
				boundaryGap: false,
				data: categories,
				axisLabel: {
					interval: Math.floor(categories.length / 6) || 0,
				}
			},
			yAxis: {
				type: 'value',
				axisLabel: {
					formatter: '{value}'
				}
			},
			series: [{
					name: '温度(℃)',
					type: 'line',
					emphasis: {
						focus: 'series'
					},
					data: tempData,
					color: '#1890FF'
				},
				{
					name: '湿度(%)',
					type: 'line',
					// smooth: true,
					emphasis: {
						focus: 'series'
					},
					data: humData,
					color: '#91CB74'
				},
				{
					name: '土壤湿度(%)',
					type: 'line',
					// smooth: true,
					emphasis: {
						focus: 'series'
					},
					data: soilData,
					color: '#FAC858'
				},
				{
					name: '雨量(%)',
					type: 'line',
					// smooth: true,
					emphasis: {
						focus: 'series'
					},
					data: rainData,
					color: '#EE6666'
				},
				{
					name: 'CO2/10(ppm)',
					type: 'line',
					// smooth: true,
					emphasis: {
						focus: 'series'
					},
					data: rainData,
					color: '#211712'
				}
			]
		};
	};

	const getUnit = (seriesName) => {
		if (seriesName.includes('℃')) return '℃';
		if (seriesName.includes('%')) return '%';
		return '';
	};

	const initOrUpdateChart = () => {
		if (!chartContainer.value) {
			console.error("Chart container not ready");
			return;
		}

		if (isLoadingHistory.value && chartInstance) {
			chartInstance.showLoading();
			return;
		}


		const option = getChartOption(historicalData.value);

		if (!chartInstance) {
			console.log("Initializing ECharts instance...");
			chartInstance = echarts.init(chartContainer.value);
			window.addEventListener('resize', handleResize);
		} else {
			chartInstance.hideLoading();
		}

		console.log("Setting chart option:", option);
		chartInstance.setOption(option, true);
	};

	let resizeTimeout;
	const handleResize = () => {
		if (resizeTimeout) clearTimeout(resizeTimeout);
		resizeTimeout = setTimeout(() => {
			chartInstance?.resize();
		}, 200);
	}

	onMounted(async () => {
		console.log('Component mounted.');

		// Fetch initial data
		await fetchLatestData();
		await fetchHistoricalData();

		nextTick(() => {
			initOrUpdateChart();
		});

		intervalId = setInterval(async () => {
			console.log('Interval: Fetching latest data and historical data for chart...');
			try {
				// Fetch data for the top cards
				await fetchLatestData();

				// **** Fetch data for the chart ****
				await fetchHistoricalData(); // **** ADD THIS LINE ****

			} catch (intervalError) {
				// Optional: Catch errors specific to the interval fetches
				// This prevents the interval from stopping if one fetch fails
				console.error("Error during periodic fetch:", intervalError);
			}

		}, 5000); // Refresh every 5000ms (5 seconds)
	});

	onUnmounted(() => {
		console.log('Component unmounted.');
		// Clear interval
		if (intervalId) {
			clearInterval(intervalId);
		}
		// Dispose chart instance and remove listener
		chartInstance?.dispose();
		window.removeEventListener('resize', handleResize);
		if (resizeTimeout) clearTimeout(resizeTimeout);
	});

	// --- Watchers ---
	// Watch for changes in historicalData to update the chart
	watch(historicalData, (newData, oldData) => {
		// Check if chartInstance exists to avoid errors during initial mount
		if (chartInstance) {
			console.log('Historical data changed, updating chart...');
			initOrUpdateChart();
		}
	}, {
		deep: true
	});
</script>

<template>
	<div class="dashboard-container">
		<h1>花卉环境监测</h1>

		<!-- Error Message -->
		<div v-if="error" class="error-message">
			错误: {{ error }}
		</div>

		<!-- *** New Grid Wrapper *** -->
		<div class="main-content-grid">

			<!-- Column 1: Current Data -->
			<div class="current-data grid-item">
				<h2>当前状态 <span v-if="isLoadingCurrent">(加载中...)</span></h2>
				<div class="sensor-cards">
					<div class="card">
						<span class="label">温度</span>
						<span class="value">{{ temperature }} ℃</span>
					</div>
					<div class="card">
						<span class="label">湿度</span>
						<span class="value">{{ humidity }} %</span>
					</div>
					<div class="card">
						<span class="label">土壤湿度</span>
						<span class="value">{{ soilValue }} %</span>
					</div>
					<div class="card">
						<span class="label">雨量</span>
						<span class="value">{{ rainValue }} %</span>
					</div>
					<div class="card">
						<span class="label">CO2/10</span>
						<span class="value">{{ Co }} ppm</span>
					</div>
				</div>
			</div>

			<!-- Column 2: Chart Area -->
			<div class="chart-area grid-item">
				<h2>历史数据图表 <span v-if="isLoadingHistory">(加载中...)</span></h2>
				<div ref="chartContainer" class="chart-container">
					<!-- ECharts will render here -->
				</div>
			</div>

			<!-- Column 3: Historical Data List -->
			<div class="historical-data grid-item">
				<h2>历史记录列表 <span v-if="isLoadingHistory && historicalData.length > 0">(加载中...)</span></h2>
				<div v-if="isLoadingHistory && historicalData.length === 0" class="loading-placeholder">
					正在加载历史数据列表，请稍候...
				</div>
				<div v-else-if="!isLoadingHistory && historicalData.length === 0 && !error" class="loading-placeholder">
					暂无历史数据记录。
				</div>
				<div v-else-if="!isLoadingHistory && historicalData.length === 0 && error && error.includes('历史数据')"
					class="loading-placeholder">
					无法加载历史数据列表。
				</div>
				<ul v-else>
					<li v-for="item in [...historicalData].reverse()" :key="item._id">
						<span class="time">{{ formatTimestamp(item.timestamp) }}</span>
						<span class="values">
							温度: {{ item.temperature }}℃,
							湿度: {{ item.humidity }}%,
							土壤: {{ item.soilValue !== undefined ? item.soilValue + '%' : 'N/A' }},
							雨量: {{ item.rainValue !== undefined ? item.rainValue + '%' : 'N/A' }},
							CO2/10: {{ item.Co }}ppm
						</span>
					</li>
				</ul>
			</div>

		</div> <!-- *** End of Grid Wrapper *** -->

	</div>
</template>

<style scoped>
	.dashboard-container {
		max-width: 1500px;
		/* Adjust as needed */
		margin: 20px auto;
		padding: 20px;
		/* Add padding around the whole container */
		font-family: Avenir, Helvetica, Arial, sans-serif;
		color: #2c3e50;
	}

	h1 {
		text-align: center;
		color: #333;
		margin-bottom: 1.5em;
		/* More space below title */
	}

	h2 {
		text-align: center;
		color: #333;
		margin-bottom: 1em;
		margin-top: 0;
		/* Remove default top margin for headings inside grid items */
		padding-bottom: 10px;
		/* Add some space below heading */
		border-bottom: 1px solid #eee;
		/* Optional separator */
	}

	h2 span {
		font-size: 0.8em;
		color: #888;
		font-weight: normal;
	}

	.error-message {
		background-color: #ffebee;
		color: #c62828;
		border: 1px solid #e57373;
		padding: 10px 15px;
		border-radius: 4px;
		margin-bottom: 20px;
		text-align: center;
	}

	/* --- Grid Layout --- */
	.main-content-grid {
		display: grid;
		grid-template-columns: 1fr 2.5fr 1.5fr;
		gap: 25px;
		/* Increase gap between columns */
		margin-top: 20px;
		/* Space below error message/title */
		align-items: start;
		/* Align items to the top of the grid row */
	}

	/* Style for items directly within the grid */
	.grid-item {
		background-color: #f8f9fa;
		border-radius: 8px;
		padding: 20px;
		/* Remove bottom margin when in grid, gap handles spacing */
		margin-bottom: 0;
		/* Allow items to potentially grow/shrink if needed, and manage internal height */
		display: flex;
		flex-direction: column;
		height: 100%;
		/* Make grid items occupy full cell height */
		box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
		/* Subtle shadow */
	}


	/* --- Column 1: Current Data --- */
	.current-data {}

	.sensor-cards {
		display: grid;
		/* Adjust card layout if needed */
		grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
		gap: 15px;
		margin-top: 15px;
	}

	.card {
		background-color: #ffffff;
		border-radius: 8px;
		padding: 15px;
		/* Slightly smaller padding */
		text-align: center;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
		border: 1px solid #e9ecef;
	}

	.card .label {
		display: block;
		font-size: 0.85em;
		/* Slightly smaller */
		color: #6c757d;
		margin-bottom: 6px;
		font-weight: 500;
	}

	.card .value {
		display: block;
		font-size: 1.4em;
		/* Slightly smaller */
		font-weight: bold;
		color: #0d6efd;
	}

	/* --- Column 2: Chart Area --- */
	.chart-area {}

	.chart-container {
		width: 100%;
		/* ** Let height be flexible, but set min-height ** */
		min-height: 400px;
		/* Ensure chart has enough vertical space */
		background-color: #fff;
		border-radius: 4px;
		border: 1px solid #e9ecef;
		flex-grow: 1;
		/* Allow chart container to fill vertical space in grid item */
	}


	/* --- Column 3: Historical Data List --- */
	.historical-data {}

	.loading-placeholder {
		text-align: center;
		color: #888;
		padding: 20px;
		background-color: #fff;
		border: 1px solid #e9ecef;
		border-radius: 4px;
		flex-grow: 1;
		/* Allow placeholder to take space */
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.historical-data ul {
		list-style: none;
		padding: 0;
		/* ** Let height be flexible within the grid item ** */
		overflow-y: auto;
		border: 1px solid #e9ecef;
		border-radius: 4px;
		background-color: #fff;
		flex-grow: 1;
		/* Allow list to fill available vertical space */
		min-height: 300px;
		/* Optional: Ensure list has some minimum height */
		/* Max-height might be needed if content pushes container too tall */
		max-height: 500px;
		/* Example max-height */
	}

	.historical-data li {
		padding: 10px 15px;
		border-bottom: 1px solid #e9ecef;
		font-size: 0.9em;
		display: flex;
		justify-content: space-between;
		flex-wrap: nowrap;
		/* Prevent wrapping if possible */
		gap: 10px;
		/* Space between time and values */
	}

	.historical-data li:last-child {
		border-bottom: none;
	}

	.historical-data li .time {
		color: #6c757d;
		font-weight: bold;
		white-space: nowrap;
		flex-shrink: 0;
		/* Prevent time from shrinking */
	}

	.historical-data li .values {
		color: #495057;
		text-align: right;
		flex-grow: 1;
		/* Allow text to wrap if necessary */
		white-space: normal;
		word-break: break-word;
		/* Break long words if needed */
	}


	/* --- Responsive Adjustments --- */

	/* Medium screens (e.g., tablets) - maybe 2 columns? */
	@media (max-width: 1200px) {

		/* Adjust breakpoint as needed */
		.main-content-grid {
			/* Stack Chart below Current Data, History List stays right or below */
			grid-template-columns: 1fr 1.5fr;
			/* Current Data | History List */
			grid-template-areas:
				"current history"
				"chart   history";
			/* Chart takes full width below current */
			/* Or simpler: 2 columns */
			/* grid-template-columns: 1fr 1fr; */
		}

		/* Assign grid areas if using template areas */
		.current-data {
			grid-area: current;
		}

		.chart-area {
			grid-area: chart;
		}

		.historical-data {
			grid-area: history;
		}

		/* Ensure chart container still has good height */
		.chart-container {
			min-height: 350px;
		}

		.historical-data ul {
			max-height: 450px;
			/* Adjust max height */
		}

	}

	/* Small screens (e.g., mobile) - stack vertically */
	@media (max-width: 767px) {
		.main-content-grid {
			grid-template-columns: 1fr;
			/* Single column */
			grid-template-areas: unset;
			/* Reset areas */
			gap: 20px;
			/* Vertical gap when stacked */
		}

		/* Reset grid area assignment */
		.current-data,
		.chart-area,
		.historical-data {
			grid-area: auto;
		}

		/* Adjust heights/margins for stacked view */
		.chart-container {
			min-height: 300px;
			/* Reduce min-height slightly */
		}

		.historical-data ul {
			max-height: 300px;
			/* Reduce list max height */
			min-height: 200px;
		}

		.grid-item {
			height: auto;
			/* Allow height to be determined by content */
			margin-bottom: 20px;
			/* Add margin back when stacked */
		}

		.grid-item:last-child {
			margin-bottom: 0;
		}

	}
</style>