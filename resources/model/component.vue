<template>
    <form :class='mapColor'>
    <div class="form-group row mt-5 ml-5">
      <div class="col-sm-5">
        <select class="form-control shadow-lg" v-model="baseValue" placeholder="請選擇匯率基準" @change='ready'>
          <option disabled selected>請選擇匯率基準</option>
          <option v-for="item in options"
              :key="item.value"
              :label="item.label"
              :value="item.value">
          </option>
        </select> 
      </div>
    </div>
  </form>
</template>

<script>
module.exports = {
  props: ['mapColor'],
  data() {
    return {
      options: [{
        value: 'USD',
        label: '美元'
      }, {
        value: 'EUR',
        label: '歐元',
      }, {
        value: 'JPY',
        label: '日圓'
      }, {
        value: 'KRW',
        label: '韓元'
      }, {
        value: 'TWD',
        label: '台幣'
      },{
        value: 'CNY',
        label: '人民幣'
      },{
        value: 'ZAR',
        label: '南非幣'
      }],
      baseValue: '',
      exchangeRateUrl: 'https://api.exchangeratesapi.io/latest',
      ratesBase: '',
      populationById: {}, // 人口數
      ratesObj: {}, // 匯率
      ratesBase: '', // 匯率基準
      renderTime: 0 
    }
  },
  mounted() {
    var format = d3.format(",");
    // Set tooltips
    this.tip = d3.tip()
                .attr('class', 'd3-tip')
                .offset([-10, 0])
                .html(function(d) {
                  let exchageRate = '';

                  if (d.exchangeRate) {
                    exchageRate = "<span class='details'><i class='fas fa-sync-alt'></i>       " + format(d.exchangeRate) + "</span>"
                  } else {
                    exchageRate = "<span class='details' style='color:red;'>N/A</span>"
                  }
                  
                  return "<strong>Country: </strong><span class='details'><i class='fa fa-flag'></i>    " + d.properties.name
                    + "<br></span>" + "<strong>Population: </strong><span class='details'><i class='fas fa-male'></i>    " + format(d.population)
                    + "<br></span>" + "<strong>Foreign Exchange Rates: </strong><span class='details'>" + exchageRate
                    +"</span>";
                });

    this.margin = {top: 0, right: 0, bottom: 0, left: 0},
                width = 1800 - this.margin.left - this.margin.right,
                height = 800 - this.margin.top - this.margin.bottom;

    this.svg = d3.select("." + this.mapColor)
                .append("svg")
                .attr("width", width)
                .attr("height", height)
                .append('g')
                .attr('class', 'map');
    
    // 2. projections: functions that convert from latitude/longitude co-ordinates to x & y co-ordinates
    this.projection = d3.geoMercator()
                       .scale(130)
                       .translate( [width / 2, height / 1.5]);
    
    // 3. Geographic path generators
    this.geoGenerator = d3.geoPath().projection(this.projection);

    this.svg.call(this.tip);

    // Initializer
    this.ready();
    
  },
  methods: {
    ready: function () {
      queue()
        .defer(d3.json, "world_countries.json")
        .defer(d3.tsv, "world_population.tsv")
        .await(this.api);
    },
    api: function (error, data, population) {
        console.log("data",data)
      let vm = this;
      let tip = vm.tip;
      let svg = vm.svg;
      let geoGenerator = vm.geoGenerator;
      let exchangeRateUrl = vm.exchangeRateUrl
      let baseValue = vm.baseValue;
      let color = vm.color;
      let mapColor = vm.mapColor;

      // 顏色級距 d3-scale
    if (mapColor == 'blue') {
      color = d3.scaleThreshold()
        .domain([10000,100000,500000,1000000,5000000,10000000,50000000,100000000,500000000,1500000000])
        .range(["rgb(247,251,255)", "rgb(222,235,247)", "rgb(198,219,239)", "rgb(158,202,225)", "rgb(107,174,214)", "rgb(66,146,198)","rgb(33,113,181)","rgb(8,81,156)","rgb(8,48,107)","rgb(3,19,43)"]);
    } else if (mapColor == 'red') {
      color = d3.scaleThreshold()
        .domain([10000,100000,500000,1000000,5000000,10000000,50000000,100000000,500000000,1500000000])
        .range(["rgb(249,224,234)", "rgb(249,224,232)", "rgb(208,52,104)", "rgb(185,31,82)", "rgb(216,88,131)", "rgb(203,23,125)","rgb(236,28,76)","rgb(198,18,60)","rgb(239,102,134)","rgb(141,26,53)"]);
    } else if (mapColor == 'green') {
      color = d3.scaleThreshold()
        .domain([10000,100000,500000,1000000,5000000,10000000,50000000,100000000,500000000,1500000000])
        .range(["rgb(183,250,170)", "rgb(105,186,89)", "rgb(76,168,58)", "rgb(44,158,21)", "rgb(34,125,16)", "rgb(26,92,13)","rgb(6,92,18)","rgb(17,77,25)","rgb(9,70,17)","rgb(4,43,9)"]);

    }

      if (baseValue) {
        exchangeRateUrl = exchangeRateUrl + "?base=" + baseValue;
      }

      axios.get(exchangeRateUrl)
          .then(
              response => {
                vm.ratesObj = response.data.rates
                vm.ratesBase = response.data.base
                }
          ).then(function () {
              population.forEach(function(d) { vm.populationById[d.id] = +d.population; });
              data.features.forEach(function(d) { d.population = vm.populationById[d.id] });
              // set the rate of exchange
              if (exchangeRateUrl) {
                data.features.forEach(function (d) {
                  if (vm.ratesBase === d.currencyCode) {
                    d.exchangeRate = 1;
                  } else {
                    d.exchangeRate = vm.ratesObj[d.currencyCode]
                  }
                });
              }

          }).then(function name(params) {
            vm.renderTime++


              svg.select(".countries").remove(); // avoid piling the g.countries elements
              svg.append("g")
                .attr("class", "countries")
                .selectAll("path")
                .data(data.features)
                .enter().append("path")
                // Create path elements and update the d attribute using the geo generator
                .attr("d", geoGenerator)
                .style("fill", function(d) { return color(vm.populationById[d.id]); })
                .style('stroke', 'white')
                .style('stroke-width', 1.5)
                .style("opacity",0.8)
              // tooltips
                .style("stroke","white")
                .style('stroke-width', 0.3)
                .on('mouseover',function(d){
                  tip.show(d);

                  d3.select(this)
                    .style("opacity", 1)
                    .style("stroke","white")
                    .style("stroke-width",3);
                })
                .on('mouseout', function(d){
                  tip.hide(d);

                  d3.select(this)
                    .style("opacity", 0.8)
                    .style("stroke","white")
                    .style("stroke-width",0.3);
                });

          // svg.append("path")
          //     .datum(topojson.mesh(data.features, function(a, b) { return a.id !== b.id; }))
          //     // .datum(topojson.mesh(data.features, function(a, b) { return a !== b; }))
          //     .attr("class", "names")
          //     .attr("d", geoGenerator);

          }).catch(function (error) { // request fail
              console.log(error);
              alert(error);
          });
    }
  }
}
</script>





<style>
  .names {
  fill: none;
  stroke: #fff;
  stroke-linejoin: round;
  }

    /* Tooltip CSS */
    .d3-tip {
    line-height: 1.5;
    font-weight: 400;
    font-family:"avenir next", Arial, sans-serif;
    padding: 6px;
    background: rgba(0, 0, 0, 0.6);
    color: #FFA500;
    border-radius: 1px;
    pointer-events: none;
    }

    /* Creates a small triangle extender for the tooltip */
    .d3-tip:after {      
      box-sizing: border-box;
      display: inline;
      font-size: 8px;
      width: 100%;
      line-height: 1.5;
      color: rgba(0, 0, 0, 0.6);
      position: absolute;
      pointer-events: none;
      
    }

    /* Northward tooltips */
    .d3-tip.n:after {
      content: "\25BC";
      margin: -1px 0 0 0;
      top: 100%;
      left: 0;
      text-align: center;
    }

    /* Eastward tooltips */
    .d3-tip.e:after {
      content: "\25C0";
      margin: -4px 0 0 0;
      top: 50%;
      left: -8px;
    }

    /* Southward tooltips */
    .d3-tip.s:after {
      content: "\25B2";
      margin: 0 0 1px 0;
      top: -8px;
      left: 0;
      text-align: center;
    }

    /* Westward tooltips */
    .d3-tip.w:after {
      content: "\25B6";
      margin: -4px 0 0 -1px;
      top: 50%;
      left: 100%;
    }

/*    text{
      pointer-events:none;
    }*/

    .details{
      color:white;
    }
</style>