<template lang="pug">
  .top
    h1 新型コロナウィルス市町村別脅威度(福井)
    .about 福井県の新型コロナウィルスの各指標の市町村別のリストです。<br>最終更新日: {{ lastUpdate }}
    v-data-table.table(
      :headers="tableHeaders",
      :items="tableData",
      :items-per-page="100",
      :hide-default-footer="true",
      :mobile-breakpoint="0"
      :sort-by.sync="sortBy"
      :sort-desc.sync="sortDesc")
    .template(v-for="header in tableHeaders")
      .grid(v-if="header.value !== 'place'")
        .grid-caption {{ header.text }}
        table
          tr(v-for="row in getGridMapData()")
            template(v-for="city in row")
              td(:style="getCellStyle(city.data, header.value)" :rowspan="city.rowspan ? city.rowspan : 1" :colspan="city.colspan ? city.colspan : 1")
                .place {{ city.name }}
                .value {{ getValue(city.data, header.value) }}
    .credit
      | Credit
      .project-home APP: 
        a(href="https://creativecommons.org/licenses/by/4.0/deed.ja" target="_new") CC BY
        | &nbsp;
        a(href="https://twitter.com/ApplePedlar") @ApplePedlar
        | &nbsp;/&nbsp;
        a(href="https://github.com/ApplePedlar/covid-19-severity-fukui" target="_new") covid-19-severity-fukui(src on GitHub)
      .data-source DATA: 
        a(href="https://creativecommons.org/licenses/by/4.0/deed.ja" target="_new") CC BY
        | &nbsp;
        a(href="https://www.pref.fukui.lg.jp/doc/toukei-jouhou/opendata/list_3.html" target="_new") 福井県 「新型コロナウイルス感染症情報」
    .links
      | Link
      .link
        a(href="https://applepedlar.github.io/covid-19-severity-jp/" target="_new") 新型コロナウィルス都道府県別脅威度
      .link
        a(href="https://applepedlar.github.io/covid-19-severity/" target="_new") 新型コロナウィルス国別脅威度

</template>

<script>

import axios from "axios"
import populations from "./populations.json"
import landarea from "./landarea.json"
import map from "./map.json"
const csvSync = require('csv-parse/lib/sync')

export default {
  data () {
    return {
      //sourceUrl: 'https://www.pref.fukui.lg.jp/doc/toukei-jouhou/covid-19_d/fil/covid19_patients.csv',
      sourceUrl: "https://covid19jp-patients.s3-ap-northeast-1.amazonaws.com/fukui.csv",// ↑がCORSポリシー違反で取れなかったのでミラーを立てた
      sortBy: 'totalPatients',
      sortDesc: true,
      tableHeaders: [
        { text: "市町村", value: "place", width: "90px", sortable: false },
        { text: "累計感染者数", value: "totalPatients" },
        { text: "10万人あたりの累計感染者数", value: "totalPatientsPerPop", sort: (a, b) => (Number(a) - Number(b)) },
        { text: "人口密度(人/平方㌖)", value: "popDensity", sort: (a, b) => (Number(a) - Number(b)) },
        { text: "宅地人口密度(人/平方㌖)", value: "residentialLandPopDensity", sort: (a, b) => (Number(a) - Number(b)) }
      ],
      tableData: [],
      populations: populations,
      landarea: landarea,
      map: map,
      lastUpdate: ""
    }
  },
  mounted () {
    document.querySelector("meta[name='viewport']").setAttribute("content", "width=800")
    this.initTableData()
    axios
      .get(this.sourceUrl)
      .then(response => {
        let source = response.data
        let csv = csvSync(source)
        let json = this.csv2json(csv)

        this.makeTableData(json)
      })
  },
  methods: {
    initTableData () {
      this.tableData = []
      Object.keys(this.populations).forEach(place => {
        this.tableData.push({
          place: place,
          population: this.populations[place],
          totalPatients: 0,
          popDensity: Math.floor(this.populations[place] / this.landarea.find(d => d.name === place)["総面積"]),
          residentialLandPopDensity: Math.floor(this.populations[place] / this.landarea.find(d => d.name === place)["宅地"])
        })
      })
    },
    makeTableData (json) {
        json.forEach(row => {
          let place = row["患者_居住地"].trim()
          let data = this.tableData.find(d => d.place === place)
          data.totalPatients++
        })

        this.tableData.forEach(data => {
          data.totalPatientsPerPop = (data.totalPatients / data.population * 100000).toFixed(3)
        })
        
        this.lastUpdate = json[json.length - 1]["公表_年月日"]
    },
    csv2json (csv) {
      var json = [];
      var csvColumns = csv[0]
      for (let i = 1; i < csv.length; i++) {
        var csvRow = csv[i]

        let jsonRow = new Object()
        for ( var colNum = 0; colNum < csvRow.length; colNum++) {
            var colData = csvRow[colNum].replace(/^['"]|['"]$/g, "");
            jsonRow[csvColumns[colNum]] = colData;
        }
        json.push(jsonRow);
      }
      return json
    },
    getGridCaption () {
      let sortByHeader = this.tableHeaders.find(header => header.value == this.sortBy)
      if (sortByHeader) {
        return sortByHeader.text
      }
    },
    getGridMapData () {
      this.map.forEach(row => {
        row.forEach(city => {
          let data = this.tableData.find(d => d.place == city.name)
          if (data) {
            city.data = data
          }
        })
      })
      return this.map
    },
    getValue (data, field) {
      if (data) {
        return data[field]
      }
    },
    getCellStyle (data, field) {
      if (!data) {
        return "border: none"
      }
      let min = this.tableData.reduce((acc, cur) => Math.min(acc ? acc : 100000000, Number(cur[field])))
      let mean = this.tableData.reduce((acc, cur) => acc + Number(cur[field]), 0) / this.tableData.length
      let value = Number(this.getValue(data, field))

      let red = 255
      let green = 255
      let blue = 255

      if (value >= mean) {
        blue = 0
        green = blue = 127 - parseInt((value - mean) / mean * 128)
        if (green < 0) {
          green = blue = 0
        }
      } else {
        green = blue = 255 - parseInt((value - min) / mean * 128)
      }

      return `background-color: rgb(${red}, ${green}, ${blue})`

    }
  },
  watch: {
    sortDesc (newSortDesc, oldSortDesc) {
      this.$nextTick(() => {
        this.sortDesc = true
      })
    }
  }
}
</script>

<style lang="sass">
.top
  max-width: 800px
  margin: 30px auto
  h1
    font-size: 24px
  .about
    margin: 20px
  .table
    max-width: 800px
    margin: 30px auto
    border: 1px silver solid
  .grid
    max-width: 500px
    margin: 30px auto
    .grid-caption
      font-weight: bold
    table
      border: 1px solid black
      width: 100%
      td
        width: 14%
        border: 1px solid black
        text-align: center
        padding: 3px 0
        &.danger
          background-color: #FF3030
        &.warn
          background-color: #FF9090
        &.alert
          background-color: #FFFFA0
        font-size: 14px
        @media (max-width: 640px)
          font-size: 12px
  .credit, .links
    font-size: 12px
    .project-home, .data-source, .link
      margin-left: 20px

</style>
