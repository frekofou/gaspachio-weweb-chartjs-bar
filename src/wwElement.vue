<template>
    <div class="chart-container" style="position: relative">
        <canvas class="chartjs-bar"></canvas>
    </div>
</template>

<script>
import {
    Chart,
    BarElement,
    BarController,
    LinearScale,
    LogarithmicScale,
    RadialLinearScale,
    TimeScale,
    TimeSeriesScale,
    Decimation,
    Filler,
    CategoryScale,
    Legend,
    Title,
    Tooltip,
    SubTitle,
} from 'chart.js';

Chart.register(
    BarElement,
    BarController,
    LinearScale,
    LogarithmicScale,
    RadialLinearScale,
    TimeScale,
    TimeSeriesScale,
    Decimation,
    Filler,
    CategoryScale,
    Legend,
    Title,
    Tooltip,
    SubTitle
);

export default {
    props: {
        content: { type: Object, required: true },
    },
    data() {
        this.chartInstance = null;
        return this.chartInstance;
    },
    computed: {
        config() {
            let labels = [];
            let datasets = [];

            if (this.content.dataType === 'guided') {
                let data =
                    (!this.content.data || Array.isArray(this.content.data)
                        ? this.content.data
                        : this.content.data.data) || [];

                const yAxis = this.content.yAxis;
                let dataXField = this.content.dataXField;
                let dataXFieldProperty = this.content.dataXFieldProperty;
                let dataYField = yAxis === 'item-count' ? this.content.dataXField : this.content.dataYField;
                let dataYFieldProperty = this.content.dataYFieldProperty;
                let groupBy = this.content.groupBy;
                let groupByProperty = this.content.groupByProperty;
                let aggregate = yAxis === 'item-count' ? 'item-count' : this.content.aggregate;
                const colors = this.content.colors;

                if (typeof data[0] !== 'object') {
                    data = data.map(value => ({ value }));
                    dataXField = 'value';
                    dataYFieldProperty = undefined;
                    dataYField = 'value';
                    dataXFieldProperty = undefined;
                    groupBy = undefined;
                    groupByProperty = undefined;
                    aggregate = 'item-count';
                }

                if (!groupBy) {
                    datasets = [
                        {
                            label: `${dataYField}`.split("['").pop().replace("']", ''),
                            backgroundColor: colors[0],
                            data: [],
                        },
                    ];
                    if (!data.length || !Array.isArray(_.get(data[0], dataXField))) {
                        datasets[0].data = data.map(item => ({
                            x: _.get(item, dataXField),
                            y: this.aggregate(
                                aggregate,
                                data
                                    .filter(elem => _.get(elem, dataXField) === _.get(item, dataXField))
                                    .map(elem => _.get(elem, dataYField))
                            ),
                        }));
                    } else {
                        const arrayValues = [
                            ...new Set(
                                data
                                    .map(item => {
                                        const result = _.get(item, dataXField, []);
                                        return Array.isArray(result)
                                            ? result.map(elem => _.get(elem, dataXFieldProperty, elem))
                                            : result;
                                    })
                                    .flat()
                            ),
                        ];
                        datasets[0].data = arrayValues.map(arrayValue => ({
                            x: arrayValue,
                            y: this.aggregate(
                                aggregate,
                                data
                                    .filter(item => {
                                        let result = _.get(item, dataXField, []);
                                        result = Array.isArray(result)
                                            ? result.map(elem => _.get(elem, dataXFieldProperty, elem))
                                            : result;
                                        return result.includes(arrayValue);
                                    })
                                    .map(item => {
                                        const result = _.get(item, dataYField, []);
                                        return Array.isArray(result)
                                            ? result.map(elem => _.get(elem, dataYFieldProperty, elem))
                                            : result;
                                    })
                            ),
                        }));
                    }
                } else {
                    const groupByValues = [
                        ...new Set(
                            data
                                .map(item => {
                                    const result = _.get(item, groupBy, []);
                                    return Array.isArray(result)
                                        ? result.map(elem => _.get(elem, groupByProperty, elem))
                                        : result;
                                })
                                .flat()
                        ),
                    ];
                    const t = [
                        ...new Set(
                            data
                                .map(item => {
                                    const result = _.get(item, dataXField, []);
                                    return Array.isArray(result)
                                        ? result.map(elem => _.get(elem, dataXFieldProperty, elem))
                                        : result;
                                })
                                .flat()
                        ),
                    ];
                    datasets = groupByValues.map((groupByValue, index) => ({
                        label: `${groupByValue}`.split("['").pop().replace("']", ''),
                        backgroundColor: colors[index % colors.length],
                        data: t.map(elem => ({
                            x: elem,
                            y: this.aggregate(
                                aggregate,
                                data
                                    .filter(item => {
                                        const currentDataXFieldValue = _.get(item, dataXField);
                                        const currentGroupByValue = _.get(item, groupBy);
                                        return (
                                            (Array.isArray(currentDataXFieldValue)
                                                ? currentDataXFieldValue
                                                      .map(elem => _.get(elem, dataXFieldProperty, elem))
                                                      .includes(elem)
                                                : currentDataXFieldValue === elem) &&
                                            (Array.isArray(currentGroupByValue)
                                                ? currentGroupByValue
                                                      .map(elem => _.get(elem, groupByProperty, elem))
                                                      .includes(groupByValue)
                                                : currentGroupByValue === groupByValue)
                                        );
                                    })
                                    .map(item => {
                                        const result = _.get(item, dataYField);
                                        return Array.isArray(result)
                                            ? result.map(elem => _.get(elem, dataYFieldProperty, elem))
                                            : result;
                                    })
                            ),
                        })),
                    }));
                }

                // Remove duplicate X values
                for (const dataset of datasets) {
                    dataset.data = dataset.data.filter(
                        (item, index) => dataset.data.findIndex(elem => item.x === elem.x) === index
                    );
                }
                // Empty values
                if (this.content.dataXEmpty === false) {
                    for (const dataset of datasets) {
                        dataset.data = dataset.data.filter(item => item.y && item.x);
                    }
                }
                // Order by
                if (this.content.dataOrderBy !== 'default') {
                    for (const item of datasets) {
                        item.data.sort((a, b) => {
                            const field = this.content.dataOrderBy;
                            return (
                                (typeof a[field] === 'string'
                                    ? a[field].localeCompare(b[field])
                                    : a[field] - b[field]) * (this.content.dataDirection === 'DESC' ? -1 : 1)
                            );
                        });
                        item.data = item.data.map(item => ({ x: `${item.x}`, y: item.y }));
                    }
                }
                labels = [...new Set(datasets.map(dataset => dataset.data.map(elem => elem.x)).flat())];
            } else {
                labels = this.content.labels;
                datasets = this.content.datasets || [];
            }

            return {
                type: 'bar',
                data: {
                    labels,
                    datasets,
                },
                options: {
                    indexAxis: this.content.axis,
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: {
                        legend: {
                            display: this.content.legendDisplay,
                            position: this.content.legendPosition,
                            align: this.content.legendAlignement,
                            labels: {
                                usePointStyle: true,
                                color: this.content.legendColor,
                                font: { size: parseInt(this.content.legendSize) },
                            },
                        },
                    },
                    interaction: {
                        intersect: false,
                    },
                    scales: {
                        x: {
                            grid: { color: this.content.gridColor, borderColor: this.content.gridColor },
                            ticks: {
                                color: this.content.ticksColor,
                                font: { size: parseInt(this.content.legendSize) },
                                display: this.content.ticksDisplay,
                                align: this.content.ticksAlign,
                                
                            },
                            stacked: this.content.stacked,
                        },
                        y: {
                            grid: { color: this.content.gridColor, borderColor: this.content.gridColor },
                            ticks: {
                                color: this.content.ticksColor,
                                font: { size: parseInt(this.content.legendSize) },
                                 display: this.content.ticksDisplay,
                                 align:this.content.ticksAlign,
                                 
                            },
                            stacked: this.content.stacked,
                            beginAtZero: this.content.startAtZero,
                        },
                    },
                },
            };
        },
    },
    watch: {
        'config.data.datasets'() {
            this.chartInstance.data.datasets = this.config.data.datasets;
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
            this.chartInstance.update();
        },
        'config.data.labels'() {
            this.chartInstance.data.labels = this.config.data.labels;
            this.chartInstance.update();
        },
         'content.legendDisplay'() {
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
        },
        'content.legendPosition'() {
            this.chartInstance.options.plugins.legend.position = this.content.legendPosition;
            this.chartInstance.update();
        },
        'content.legendAlignement'() {
            this.chartInstance.options.plugins.legend.align = this.content.legendAlignement;
            this.chartInstance.update();
        },
        'content.legendColor'() {
            this.chartInstance.options.plugins.legend.labels.color = this.content.legendColor;
            this.chartInstance.update();
        },
        'content.legendSize'() {
            this.chartInstance.options.plugins.legend.labels.font.size = parseInt(this.content.legendSize);
            this.chartInstance.options.scales.x.ticks.font.size = parseInt(this.content.legendSize);
            this.chartInstance.options.scales.y.ticks.font.size = parseInt(this.content.legendSize);
            this.chartInstance.update();
        },
        'content.gridColor'() {
            this.chartInstance.options.scales.x.grid.borderColor = this.content.gridColor;
            this.chartInstance.options.scales.x.grid.color = this.content.gridColor;
            this.chartInstance.options.scales.y.grid.borderColor = this.content.gridColor;
            this.chartInstance.options.scales.y.grid.color = this.content.gridColor;
            this.chartInstance.update();
        },
         'content.ticksColor'() {
            this.chartInstance.options.scales.x.ticks.color = this.content.ticksColor;
            this.chartInstance.options.scales.y.ticks.color= this.content.ticksColor;
            this.chartInstance.update();
        },
        'content.axis'() {
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
        },
        'content.ticksDisplay'() {
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
        },
        'content.ticksAlign'() {
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
        },
         'content.ticksColor'() {
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
        },
        'content.stacked'() {
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
        },
        'content.startAtZero'() {
            if (this.chartInstance) this.chartInstance.destroy();
            this.initChart();
        },
        'content.dataYField'() {
            // eslint-disable-next-line vue/require-explicit-emits
            this.$emit('update:content:effect', { dataYFieldProperty: null });
        },
        'content.dataYFieldProperty'() {
            const data =
                (!this.content.data || Array.isArray(this.content.data) ? this.content.data : this.content.data.data) ||
                [];
            let field = _.get(data[0], this.content.dataYField);
            const isArray = Array.isArray(field);
            if (Array.isArray(field) && field.length)
                field = _.get(field[0], this.content.dataYFieldProperty, field[0]);
            const isNumber = Number.isFinite(data[0] && this.content.dataYField && field);
            if (isNumber) {
                // eslint-disable-next-line vue/require-explicit-emits
                this.$emit('update:content:effect', { aggregate: isArray ? 'sum' : 'value' });
            }
        },
        'content.dataXField'() {
            // eslint-disable-next-line vue/require-explicit-emits
            this.$emit('update:content:effect', { dataXFieldProperty: null });
        },
        'content.groupBy'() {
            // eslint-disable-next-line vue/require-explicit-emits
            this.$emit('update:content:effect', { groupByProperty: null });
        },
    },
    mounted() {
        this.initChart();
    },
    beforeUnmount() {
        this.chartInstance.destroy();
    },
    methods: {
        initChart() {
            const element = this.$el.querySelector('.chartjs-bar');
            this.chartInstance = new Chart(element, this.config);
        },
        aggregate(operator, data) {
            if (!data) return undefined;
            switch (operator) {
                case 'item-count':
                    return [data].flat().length;
                case 'distinct':
                    return [...new Set([data].flat())].length;
                case 'value':
                    return [data].flat()[0];
                case 'sum':
                    return this.sum([data].flat());
                case 'average':
                    return this.average([data].flat());
                case 'median':
                    return this.median([data].flat());
                case 'min':
                    return Math.min(...[data].flat());
                case 'max':
                    return Math.max(...[data].flat());
            }
        },
        median(arr) {
            const mid = Math.floor(arr.length / 2),
                nums = [...arr].sort((a, b) => a - b);
            return arr.length % 2 !== 0 ? nums[mid] : (nums[mid - 1] + nums[mid]) / 2;
        },
        sum(arr) {
            return arr.reduce((a, b) => a + b, 0);
        },
        average(arr) {
            return arr.reduce((a, b) => a + b, 0) / arr.length;
        },
    },
};
</script>

<style lang="scss" scoped>
.chart-container {
    width: inherit;
    height: inherit;
    #chartjs-vertical-bar {
        width: 100% !important;
        height: 100% !important;
    }
}
</style>
