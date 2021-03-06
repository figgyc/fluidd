<script lang="ts">
import { Vue, Component, Prop, Mixins, Watch } from 'vue-property-decorator'
import Chart from 'chart.js'
import { Line, mixins } from 'vue-chartjs'
import 'chartjs-plugin-colorschemes'

@Component({})
export default class TemperatureChartWidget extends Mixins(Line, mixins.reactiveProp) {
  @Prop({ required: true, default: {} })
  public chartData!: Chart.ChartData

  @Prop({ default: false })
  public resetChart!: boolean;

  @Prop()
  public chartOptions!: Chart.ChartOptions;

  private options: Chart.ChartOptions = {};
  private chart!: Chart;
  private chartTimer = -1;

  @Watch('chartData')
  onChartDataChange () {
    this.updateChart()
  }

  mounted () {
    if (!this.chartOptions) {
      this.applyDefaultOptions()
    }

    // Draw chart
    this.renderChart(this.chartData, this.options)
    this.chart = this.$data._chart

    // Update chart
    // this.chartTimer = setInterval(this.updateChart, 1000)
  }

  private getXTicks () {
    const maxTick = new Date()
    const minTick = new Date().setMinutes(maxTick.getMinutes() - 10)
    return {
      min: minTick,
      max: maxTick
    }
  }

  private updateChart () {
    const ticks = this.getXTicks()
    if (
      this.chart &&
      this.chart.config &&
      this.chart.config.options &&
      this.chart.config.options.scales &&
      this.chart.config.options.scales.xAxes &&
      this.chart.config.options.scales.xAxes.length &&
      this.chart.config.options.scales.xAxes[0].ticks &&
      this.chart.config.options.scales.yAxes &&
      this.chart.config.options.scales.yAxes.length &&
      this.chart.config.options.scales.yAxes[0].ticks
    ) {
      this.chart.config.options.scales.xAxes[0].ticks.min = ticks.min
      this.chart.config.options.scales.xAxes[0].ticks.max = ticks.max
      this.chart.config.options.scales.yAxes[0].ticks.max = 300
      // this.chart.update()
    }
  }

  private applyDefaultOptions () {
    const ticks = this.getXTicks()
    this.options.maintainAspectRatio = false
    this.options.responsive = true
    this.options.scales = {
      xAxes: [
        {
          gridLines: {
            color: '#333333'
          },
          type: 'time',
          time: {
            // parser: 'MM/DD/YYYY HH:mm',
            unit: 'minute',
            displayFormats: {
              minute: 'YYYY-MM-DDTHH:mm:ssZ'
            }
          },
          ticks: {
            min: ticks.min,
            max: ticks.max,
            callback: (value) => {
              return Vue.$dayjs(value, { locale: 'en-chart' }).fromNow(true)
            }
          }
        }
      ],
      yAxes: [
        {
          gridLines: {
            color: '#333333'
          },
          ticks: {
            min: 0,
            max: 300,
            stepSize: 60,
            callback: (value) => {
              return value + '°C'
            }
          }
        }
      ]
    }

    this.options.plugins = {
      colorschemes: {
        scheme: 'tableau.GoldPurple7',
        fillAlpha: 0.05
      }
    }

    this.options.elements = {
      point: {
        radius: 0
      },
      line: {
        tension: 0
      }
    }

    this.options.legend = {
      labels: {
        filter: (item) => {
          if (item && item.text) {
            return !item.text.includes('Target')
          }
        }
      }
    }

    this.options.tooltips = {
      enabled: false,
      mode: 'x',
      intersect: false,
      position: 'nearest',
      titleAlign: 'left'
    }

    this.options.hover = {
      mode: undefined
    }
  }
}

</script>
