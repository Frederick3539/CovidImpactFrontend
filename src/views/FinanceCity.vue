<template>
    <section class="city">
        <h1 v-if="historyDataStockError">
            No historical stock data found for {{ stockName }}
        </h1>

        <h1>Currently only supports USA Counties</h1>
        <div
            class="select--margin--top"
            v-if="!loadingCounties && !countiesError"
        >
            <Select
                :data="countyList"
                :placeholder="`${
                    countyList[countySelected].name || 'Choose county'
                }`"
                @newSelection="countyChange"
            />
        </div>
        <Loading v-if="loadingCounties" />
        <h1 v-if="countiesError">Error ocuured while getting Counties</h1>

        <div v-if="!loadingCounty && !countyError" class="select--margin--top">
            <Select
                :data="countyNameList"
                placeholder="Choose county"
                @newSelection="countyNameChange"
            />
        </div>

        <Loading v-if="loadingCounty" />
        <h1 v-if="countyError">Error ocuured while getting County</h1>
    </section>
</template>

<script>
import Select from "../components/Select";
import Loading from "../components/Loading";

// @group Views
/**
 * To analyse the impact of the COVID-19 citywise
 */
export default {
    components: {
        Select,
        Loading,
    },
    props: {
        // The theme for the page
        theme: { type: String, required: false, default: "light" },
        // Stock for which the data is to fetched.
        stockName: {
            type: Object,
            required: false,
            deafult: () => ({
                isEnabled: true,
                name: "Apple Inc.",
                cik: "320193",
                type: "cs",
                symbol: "AAPL",
                region: "US",
                currency: "USD",
                exchange: "NAS",
                queryableSymbol: "aapl",
                iexId: "IEX_4D48333344362D52",
                figi: "BBG000B9XRY4",
                date: "2020-11-18",
                queryable: "apple inc.",
            }),
        },
    },
    data: function () {
        return {
            loadingCounties: true,
            countiesError: false,
            countyList: [],
            countySelected: 0,
            countyError: false,
            loadingCounty: true,
            currentCountyData: [],
            countyNameList: [],
            historyDataStockError: false,
            dataStock: {},
        };
    },
    methods: {
        /**
         * @vuese
         * Zips two given arrays into a two dimensional array.
         * @arg array to zip
         * @arg array to zip
         */
        zip: (a, b) => a.map((k, i) => [Date.parse(b[i]), k]),
        /**
         * @vuese
         * Gets a list of the counties for which data can be fetched.
         */
        getCountiesList: async function () {
            try {
                this.countiesError = false;
                this.loadingCounties = true;
                const response = await fetch(
                    "https://disease.sh/v3/covid-19/historical/usacounties"
                );
                const data = await response.json();
                const countyList = data.map((element, index) => {
                    return {
                        name: element,
                        code: index,
                    };
                });

                this.countyList = [...countyList];
                this.loadingCounties = false;
            } catch (error) {
                console.log(error);
                this.loadingCounties = false;
                this.countiesError = true;
            }
        },
        /**
         * @vuese
         * Get the data for the selected county. Default is alphabetically.
         */
        getCountyData: async function () {
            try {
                this.countyError = false;
                this.loadingCounty = true;
                const response = await fetch(
                    `https://disease.sh/v3/covid-19/historical/usacounties/${
                        this.countyList[this.countySelected].name
                    }?lastdays=365`
                );
                const data = await response.json();
                this.currentCountyData = [...data];

                const countyNameList = data.map((element, index) => {
                    return {
                        name: element.county,
                        code: index,
                    };
                });

                this.countyNameList = [...countyNameList];

                this.loadingCounty = false;
            } catch (error) {
                console.log(error);
                this.loadingCounty = false;
                this.countyError = true;
            }
        },
        /**
         * @vuese
         * Get the data for the selected stock.
         */
        getStockData: async function () {
            try {
                const responseFinance = await fetch(
                    `https://sandbox.iexapis.com/stable/stock/CCL/chart/1y?token=${process.env.VUE_APP_IEX_TOKEN}`
                );
                const dataFinance = await responseFinance.json();

                let dates = [];
                let close = [];
                let open = [];
                let high = [];
                let low = [];

                dataFinance.forEach((element) => {
                    dates.push(element.date);
                    close.push(element.close);
                    open.push(element.open);
                    high.push(element.high);
                    low.push(element.low);
                });

                const series = [
                    {
                        name: "Open",
                        data: this.zip(open, dates),
                        color: "green",
                    },
                    {
                        name: "Close",
                        data: this.zip(close, dates),
                        color: "blue",
                    },
                    {
                        name: "High",
                        data: this.zip(high, dates),
                        color: "yellow",
                    },
                    {
                        name: "Low",
                        data: this.zip(low, dates),
                        color: "red",
                    },
                ];

                this.dataStock = { series };
            } catch (error) {
                this.historyDataStockError = true;
            }
        },
        /**
         * @vuese
         * Change the state for getting new counties/cities.
         */
        countyChange: function (county) {
            this.countySelected = county[1];
            this.getCountyData();
        },
        /**
         * @vuese
         * Get the selected county.
         */
        countyNameChange: function (county) {
            this.countySelected = county[1];
        },
    },
    async mounted() {
        await this.getStockData();
        await this.getCountiesList();
        await this.getCountyData();
    },
    watch: {
        stockName: async function () {
            await this.getStockData();
            await this.getCountiesList();
            await this.getCountyData();
        },
    },
};
</script>

<style>
.city {
    width: 100%;
    height: 100%;
}

.select--margin--top {
    margin-top: 20px;
}
</style>