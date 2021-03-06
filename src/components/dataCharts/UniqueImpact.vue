<template>
    <Chart
      xKey="time"
      yKey="value"
      :title="ChartTitle"
      :info="ChartInfo"
      :source="source"
      :chartNote="chartNote"
      :xAxisNote="xAxisNote"
      :data="ChartData"
      :xFormat="xFormat"
    />
</template>

<script>
import Chart from "../charts/AnimatedLineChart";

import { json } from "d3-fetch";
import { timeParse } from "d3-time-format";
import { max, maxIndex, min, minIndex, bisector } from "d3-array";
import { fredUnits, template } from "../functions/d3-max";
import { format } from "d3-format";

import recessionsObj from "../functions/recessionDates";

//PCEPI
const apiKey = "84e147c6a80307d0560aca7eaa8a146a"; //GOOD THING I AM NOT PUBLICALLY HOSTING THIS ON GITHUB, OTHERWISE THIS WOULD BE PRETTY DUMB
const set1 = "PAYEMS";
const fUnits = "lin";
const start = "1940-01-01"; //YYYY-MM-DD
const parseTime = timeParse("%Y-%m-%d");
const parseRecessions = timeParse("%Y - %m - %d");

const _MS_PER_DAY = 1000 * 60 * 60 * 24;

function dateDiffInDays(a, b) {
  // Discard the time and time-zone information.
  const utc1 = Date.UTC(a.getFullYear(), a.getMonth(), a.getDate());
  const utc2 = Date.UTC(b.getFullYear(), b.getMonth(), b.getDate());

  return Math.floor((utc2 - utc1) / _MS_PER_DAY);
}

function dataSeries(set) {
  return `https://maxpod-corsfixed.herokuapp.com/https://api.stlouisfed.org/fred/series/observations?series_id=${set}&api_key=${apiKey}&observation_start=${start}&units=${fUnits}&file_type=json&callback=?`;
}

function dataInfo(set) {
  return `https://maxpod-corsfixed.herokuapp.com/https://api.stlouisfed.org/fred/series?series_id=${set}&api_key=${apiKey}&observation_start=${start}&units=${fUnits}&file_type=json`;
}

export default {
  name: "UniqueImpact",
  data: () => ({
    source: "See FRED (Federal Reserve St. Louis, 2020)",
    chartNote: "% Job Losses Compared to Prior Employment Peak",
    xAxisNote: "Days Until Recovered",
    ChartTitle: "Job losses during post-war recessions",
    recessionsObj: recessionsObj,
    ChartData: [],
    ChartInfo: {},
    xFormat() {
      return format(",.0f")
    },
  }),
  mounted() {  
    let recessionsObj = this.recessionsObj.slice();

    Promise.all([json(dataSeries(set1)), json(dataInfo(set1))])
      .then((responses) => {
        let dataSet = [];
        let values = responses[0].observations;
        let splicedValues = [];

        recessionsObj[recessionsObj.length - 1].trough =
          values[values.length - 1].date;

        //Define indexTrough
        recessionsObj.forEach((recession, index) => {
          let indexTrough = values
            .map((element) => {
              return element.date;
            })
            .indexOf(recession.trough);

          recession.indexTrough = indexTrough;
        });

        //Use indexTrough
        recessionsObj.forEach((recession, index) => {
          if (index == 0) {
            let slice = values.slice(0, recessionsObj[index + 1].indexTrough);

            splicedValues.push({
              key: recession.key,
              values: slice,
            });
          } else {
            let slice;

            if (index + 1 < recessionsObj.length) {
              slice = values.slice(
                recessionsObj[index].indexTrough,
                recessionsObj[index + 1].indexTrough
              );
            } else {
              slice = values.slice(
                recessionsObj[index].indexTrough,
                values.length
              );
            }

            let priorValues = splicedValues[index - 1].values,
              priorMaxIndex = maxIndex(priorValues, (d) => {
                return d.value;
              }),
              priorMax = splicedValues[index - 1].values[priorMaxIndex].value,
              priorDate = splicedValues[index - 1].values[priorMaxIndex].date;

            let popStartIndex = values
              .map((value) => {
                return value.date;
              })
              .indexOf(priorDate);

            let recovIndex = minIndex(slice, (d) => {
              return Math.abs(d.value - priorMax);
            });

            let popEndIndex;
            if (!slice[recovIndex]) {
              popEndIndex = values.length;
            } else {
              popEndIndex = values
                .map((value) => {
                  return value.date;
                })
                .indexOf(slice[recovIndex].date);
            }

            splicedValues.push({
              key: recession.key,
              priorKey: splicedValues[index - 1].key,
              priorMax,
              priorDate,

              values: slice,
              popStartIndex,
              popEndIndex,
            });
          }
        });

        //console.log("recessionObj: ", recessionsObj,"splicedvalues: ",splicedValues)
        let recessionIndex = 1;
        splicedValues.forEach((element, index) => {
          if (index == 0) {
            return;
          }
          if (element.key == 1990) {
            return;
          }

          let slice = values.slice(element.popStartIndex, element.popEndIndex);

          slice.forEach((sliceValues) => {
            dataSet.push({
              value: (sliceValues.value-element.priorMax)/element.priorMax,
              time: dateDiffInDays(parseTime(element.priorDate), parseTime(sliceValues.date)),
              key: element.key,
            });
          });
        });

        let { notes, last_updated, id, units, title } = responses[1].seriess[0];
        last_updated = parseTime(last_updated.split(" ")[0]);

        this.ChartInfo = { notes, last_updated, id, units };
        this.ChartData = dataSet;
      })
      .catch((error) => {
        console.warn(error);
      });
  },
  components: {
    Chart,
  },
};
</script>
