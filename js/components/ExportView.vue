<style lang="scss">
@import '~bootstrap/scss/functions';
@import '~bootstrap/scss/variables';
@import '~bootstrap/scss/mixins';

.card-columns {
    column-count: 1;

    @include media-breakpoint-only(md) {
        column-count: 2;
    }

    @include media-breakpoint-only(lg) {
        column-count: 3;
    }

    @include media-breakpoint-up(xl) {
        column-count: 4;
    }
}
</style>

<template>
    <div>
        <b-alert show>
            This feature is highly experimental. Many features are a work in progress and may be disabled.
        </b-alert>
        <b-alert show>
            Make sure your seed is not empty!
        </b-alert>
        <b-form class='border rounded p-3'>
            <h5>Export Options</h5>
            <b-form-group label="Export Date Range" description="The range of dates this export will include (inclusive)."
                label-cols-sm='4'>
                <label for="start-date-picker">Start Date</label>
                <b-form-datepicker id="start-date-picker" :max="endDate" :min="new Date(2000, 0, 1)" v-model="startDate"
                    class="mb-2" value-as-date></b-form-datepicker>
                <label for="end-date-picker">End Date</label>
                <b-form-datepicker id="end-date-picker" :min="startDate" :max="new Date(2060, 11, 31)" v-model="endDate"
                    class="mb-2" value-as-date></b-form-datepicker>
            </b-form-group>
            <b-form-group label="Export Inclusions"
                description="All features you wish to be in the export. Certain selections may result in multiple files generated."
                label-cols-sm='4'>
                <b-form-checkbox-group v-model="selectedFeatures" :options="FEATURES">
                </b-form-checkbox-group>
            </b-form-group>
            <b-form-group label="Export Format" description="The file format of the export(s)." label-cols-sm='4'>
                <b-form-select v-model="selectedFormat" :options="FORMATS"></b-form-select>
            </b-form-group>
            <div class="text-center">
                <b-button variant="primary" :disabled="progress >= 0" @click='exportData'>Export</b-button>
            </div>
            <b-progress :value="progress" max="100" animated v-show="progress >= 0" class="m-3"></b-progress>
        </b-form>
    </div>
</template>

<script lang='ts'>
import { Vue, Component, Prop } from 'vue-property-decorator'
import { Forecast, Weather } from '../model';
import { kebabToTitle } from '../utils';
import { SpecialCloud } from '../../pkg/index';

const FEATURES = [
    {
        value: "weather", text: "Weather"
    },
    {
        value: "aurora-borealis", text: "Aurora Borealis (WIP)", disabled: true
    },
    {
        value: "meteor-showers", text: "Meteor Showers (WIP)", disabled: true
    }
] as const

const FORMATS = [
    {
        value: "csv", text: "CSV", mime: "text/csv"
    },
    {
        value: "ics", text: "ICS (iCal) [NOT YET SUPPORTED]", mime: "text/calendar", disabled: true
    }
] as const

type FORMAT_TYPES = typeof FORMATS[number]["value"];
type MIME_TYPES = typeof FORMATS[number]["mime"];

const MIMES: { [k in FORMAT_TYPES]: MIME_TYPES } = FORMATS.reduce((map, obj) => { map[obj.value] = obj.mime; return map; }, {} as any)

@Component
export default class ExportView extends Vue {
    readonly FEATURES = FEATURES
    readonly FORMATS = FORMATS
    kebabToTitle = kebabToTitle

    startDate: Date = new Date(new Date(new Date().setFullYear(new Date().getFullYear() - 1)).toDateString())
    endDate: Date = new Date(new Date().toDateString())
    selectedFeatures: typeof FEATURES[number]["value"][] = []
    selectedFormat: typeof FORMATS[number]["value"] = "csv"
    progress: number = -1
    progressIntervalID: NodeJS.Timeout | null = null;
    @Prop(Forecast) readonly forecast!: Forecast

    exportData() {
        this.setProgress(10)

        var data: { [key: string]: Blob } = {}
        // Grab all data and populate data dict
        for (var i = 0; i < this.selectedFeatures.length; i++) {
            var internalForecast = new Forecast(this.forecast.island)
            while (internalForecast.year > this.startDate.getFullYear()) {
                internalForecast.setPreviousYear();
            }
            while (internalForecast.month - 1 > this.startDate.getMonth()) {
                internalForecast.setPreviousMonth();
            }
            var name = this.forecast.islandName ?? "unnamed_island";
            var start = this.startDate.toDateString().split(" ").join("_");
            var end = this.endDate.toDateString().split(" ").join("_");
            var fileName = `meteonook_export_${name}_from_${start}_to_${end}_${this.selectedFeatures[i]}.${this.selectedFormat}`

            if (this.selectedFormat == "ics") {
                alert("Export Type (ICS/iCal) not supported / WIP")
                break;
            };

            if (this.selectedFeatures[i] == "weather") {
                data[fileName] = this.generateWeatherBlob(internalForecast, MIMES[this.selectedFormat])
            } else if (this.selectedFeatures[i] == "aurora-borealis") {
                alert("MeteoNook Feature (Aurora Borealis) not support supported / WIP")
            } else if (this.selectedFeatures[i] == "meteor-showers") {
                alert("MeteoNook Feature (Meteor Showers) not support supported / WIP")
            }
            this.setProgress(10 + (i + 1) / this.selectedFeatures.length * 80);
        }

        setTimeout(() => {
            var i = 0;
            //Save Files
            for (var key in data) {
                const link = document.createElement('a');
                link.href = URL.createObjectURL(data[key]);
                link.download = key;
                link.click();
                URL.revokeObjectURL(link.href);
                // link.remove();
                this.setProgress(85 + i / Object.keys(data).length)
            }
            this.setProgress(100)
            this.resetSubmitUi()
        }, 1000)
    }

    generateWeatherBlob(internalForecast: Forecast, mime: typeof MIMES[FORMAT_TYPES]): Blob {
        var data = ""
        var dateFormatter = new Intl.DateTimeFormat("en-US", {
            month: "2-digit",
            day: "2-digit",
            year: "numeric"
        });

        var timeFormatter = new Intl.DateTimeFormat("en-US", {
            hour: "2-digit",
            minute: "2-digit"
        })
        if (mime == "text/csv") {
            data = "Date,Time,Wind Power,Weather,Special Cloud,Any Fog,Water Fog,Heavy Fog,Rainbow Count\n"
            while (internalForecast.year < this.endDate.getFullYear() || internalForecast.month - 1 <= this.endDate.getMonth()) {
                var currMonth = internalForecast.currentMonth;
                for (var day of currMonth.days) {
                    if (day.date > this.endDate) break;

                    var prefix = dateFormatter.format(day.date)
                    for (var i = 0; i < day.weather.length; i++) {
                        var timestamp = new Date(day.date);
                        timestamp.setHours(i);
                        timestamp = internalForecast.add5hr(timestamp);
                        var windPower = day.windPower[i];
                        var weather = Weather[day.weather[i]];
                        var specialCloudInfo = SpecialCloud[day.specialClouds[i]]
                        var waterFog = i == 2 && day.waterFog ? "TRUE" : "FALSE";
                        var heavyFog = i == 2 && day.heavyFog ? "TRUE" : "FALSE";
                        var anyFog = i == 2 && (day.waterFog || day.heavyFog) ? "TRUE" : "FALSE";
                        var rainbowCount = i + 5 == day.rainbowHour ? day.rainbowCount : 0;
                        data += `${prefix},${timeFormatter.format(timestamp)},${windPower},${weather},${specialCloudInfo},${anyFog},${waterFog},${heavyFog},${rainbowCount}\n`
                        prefix = ""
                    }
                }
                internalForecast.setNextMonth();
            }
        }

        return new Blob([data], {
            type: mime
        })
    }

    setProgress(progressDesired: number) {
        if (this.progressIntervalID) {
            clearInterval(this.progressIntervalID)
        }
        progressDesired = Math.trunc(progressDesired)
        if (progressDesired < 0) progressDesired = 0;
        if (progressDesired > 100) progressDesired = 100;

        this.progressIntervalID = setInterval(() => {
            if (progressDesired == 100) {
                this.progress = progressDesired;
            } else {

                this.progress = 0.95 * this.progress + 0.05 * progressDesired;
            }
        }, 100);
    }

    resetSubmitUi() {
        if (this.progressIntervalID) clearInterval(this.progressIntervalID);
        this.progress = -1;
    }
}
</script>