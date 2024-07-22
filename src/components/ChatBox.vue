<template>
  <div class="chat-container">
    <h1>100 ATL Homes</h1>
    <textarea
      v-model="userInput"
      placeholder="Query your ATL home data..."
    ></textarea>
    <button @click="getResponse">Submit</button>
    <div v-if="conversationHistory.length">
      <h2>Conversation History:</h2>
      <ul>
        <div v-for="(conversation, index) in conversationHistory" :key="index">
          <strong>You:</strong> {{ conversation.user }}<br />
        </div>
      </ul>
    </div>
    <div v-if="response">
      <h2>Response:</h2>
    </div>
    <canvas id="chart" v-if="chartData"></canvas>
  </div>
</template>

<script>
import { toRaw } from "vue";
import axios from "axios";
import {
  Chart,
  BarController,
  BarElement,
  PieController,
  CategoryScale,
  LinearScale,
  Title,
  Tooltip,
  Legend,
  ArcElement,
} from "chart.js";

const apiKey = process.env.VUE_APP_GPT_API_KEY;

// Register Chart.js components
Chart.register(
  BarController,
  BarElement,
  PieController,
  CategoryScale,
  LinearScale,
  Title,
  Tooltip,
  Legend,
  ArcElement
);
export default {
  name: "ChatBox",
  data() {
    return {
      userInput: "",
      response: null,
      chartData: null,
      housingData: [],
      chart: null,
      conversationHistory: [],
    };
  },
  methods: {
    async getResponse() {
      try {
        const conversationHistoryString = this.conversationHistory
          .map(
            (conversation) =>
              `You: ${conversation.user}\nChatGPT: ${conversation.response}`
          )
          .join("\n\n");
        const res = await axios.post(
          "https://api.openai.com/v1/chat/completions",
          {
            model: "gpt-3.5-turbo",
            messages: [
              {
                role: "system",
                content:
                  "You are an assistant that helps generate data configurations for visualizations. You have access to a dataset of 100 houses in Atlanta. Each house has the following properties: id, address, neighborhood, zipCode, mostRecentSalePrice, sqft, yearBuilt, bedrooms, bathrooms, garageSpaces, lotSize. Your task is to interpret user queries about this data and return a JSON object with a 'dataConfig' that specifies the type of chart, filters to apply, grouping criteria, colors, and chart options.",
              },
              {
                role: "user",
                content:
                  'Here is an example of the structure of a single house record:\n{\n  "id": 95,\n  "address": "9292 Maple Dr",\n  "neighborhood": "Decatur",\n  "zipCode": "30030",\n  "mostRecentSalePrice": 400000,\n  "sqft": 1900,\n  "yearBuilt": 1940,\n  "bedrooms": 3,\n  "bathrooms": 2,\n  "garageSpaces": 1,\n  "lotSize": 0.25\n}\n\nWhen given a query, you should return a JSON object with a \'dataConfig\' property at the top level that looks like this example:\n{\n  "type": "pie",\n  "filters": {\n    "mostRecentSalePrice": ">200000"\n  },\n  "groupBy": "neighborhood",\n  "colors": ["#FF6384", "#36A2EB", "#FFCE56"],\n  "options": {\n    "responsive": true\n  }\n}\n',
              },
              {
                role: "user",
                content: `Here is your conversation history with the user (if it exists yet): ${conversationHistoryString}. Again please ensure that your responses are a JSON object with a 'dataConfig' property at the top level that looks like this example:{  "type": "pie",  "filters": {    "mostRecentSalePrice": "< 500000"  },  "groupBy": "price",\n  "colors": ["#FF6384", "#36A2EB", "#FFCE56"],  "options": {    "responsive": true  }}`,
              },
              { role: "user", content: this.userInput },
            ],
            max_tokens: 150,
          },
          {
            headers: {
              Authorization: `Bearer ${apiKey}`,
              "Content-Type": "application/json",
            },
          }
        );
        this.response = res.data.choices[0].message.content;
        const parsedResponse = JSON.parse(this.response);
        try {
          this.processResponse(parsedResponse.dataConfig);
          this.conversationHistory.push({
            user: this.userInput,
            response: this.response,
          });
        } catch (processError) {
          console.log("processError ===>:", processError);
        }
      } catch (error) {
        console.error("Error fetching response:", error);
        this.response = "Error fetching response.";
      }
    },
    async fetchHousingData() {
      try {
        const response = await axios.get("/homes.json");
        this.housingData = response.data;
      } catch (error) {
        console.error("Error fetching housing data:", error);
      }
    },
    processResponse(config) {
      const filteredData = this.filterData(config.filters);
      this.chartData = this.prepareChartData(filteredData, config.groupBy);
      this.$nextTick(() => {
        this.renderChart(config);
      });
    },
    filterData(filters) {
      if (filters == null) {
        return this.housingData;
      }
      return this.housingData.filter((hm) => {
        const home = toRaw(hm);
        let include = true;
        if (!home || home === undefined) {
          return true;
        }

        if (
          filters.mostRecentSalePrice &&
          home?.mostRecentSalePrice !== undefined
        ) {
          const priceNum = parseFloat(
            filters.mostRecentSalePrice.replace(/[^0-9.]/g, "")
          );
          if (
            filters.mostRecentSalePrice.includes(">") &&
            !(home?.mostRecentSalePrice > priceNum)
          ) {
            include = false;
          }
          if (
            filters.mostRecentSalePrice.includes("<") &&
            !(home?.mostRecentSalePrice < priceNum)
          ) {
            include = false;
          }
          if (
            filters.mostRecentSalePrice.includes(">=") &&
            !(home?.mostRecentSalePrice >= priceNum)
          ) {
            include = false;
          }
          if (
            filters.mostRecentSalePrice.includes("<=") &&
            !(home?.mostRecentSalePrice <= priceNum)
          ) {
            include = false;
          }
        }

        if (filters.sqft && home?.sqft !== undefined) {
          const sqftNum = parseFloat(filters.sqft.replace(/[^0-9.]/g, ""));
          if (filters.sqft.includes(">") && !(home?.sqft > sqftNum)) {
            include = false;
          }
          if (filters.sqft.includes("<") && !(home?.sqft < sqftNum)) {
            include = false;
          }
          if (filters.sqft.includes(">=") && !(home?.sqft >= sqftNum)) {
            include = false;
          }
          if (filters.sqft.includes("<=") && !(home?.sqft <= sqftNum)) {
            include = false;
          }
        }

        if (filters.bedrooms && home?.bedrooms !== undefined) {
          const bedroomsNum = parseInt(
            filters.bedrooms.replace(/[^0-9]/g, ""),
            10
          );
          if (
            filters.bedrooms.includes(">") &&
            !(home?.bedrooms > bedroomsNum)
          ) {
            include = false;
          }
          if (
            filters.bedrooms.includes("<") &&
            !(home?.bedrooms < bedroomsNum)
          ) {
            include = false;
          }
          if (
            filters.bedrooms.includes(">=") &&
            !(home?.bedrooms >= bedroomsNum)
          ) {
            include = false;
          }
          if (
            filters.bedrooms.includes("<=") &&
            !(home?.bedrooms <= bedroomsNum)
          ) {
            include = false;
          }
        }

        if (filters.bathrooms && home?.bathrooms !== undefined) {
          const bathroomsNum = parseFloat(
            filters.bathrooms.replace(/[^0-9.]/g, "")
          );
          if (
            filters.bathrooms.includes(">") &&
            !(home?.bathrooms > bathroomsNum)
          ) {
            include = false;
          }
          if (
            filters.bathrooms.includes("<") &&
            !(home?.bathrooms < bathroomsNum)
          ) {
            include = false;
          }
          if (
            filters.bathrooms.includes(">=") &&
            !(home?.bathrooms >= bathroomsNum)
          ) {
            include = false;
          }
          if (
            filters.bathrooms.includes("<=") &&
            !(home?.bathrooms <= bathroomsNum)
          ) {
            include = false;
          }
        }

        if (filters.yearBuilt && home?.yearBuilt !== undefined) {
          const yearBuiltNum = parseInt(
            filters.yearBuilt.replace(/[^0-9]/g, ""),
            10
          );
          if (
            filters.yearBuilt.includes(">") &&
            !(home?.yearBuilt > yearBuiltNum)
          ) {
            include = false;
          }
          if (
            filters.yearBuilt.includes("<") &&
            !(home?.yearBuilt < yearBuiltNum)
          ) {
            include = false;
          }
          if (
            filters.yearBuilt.includes(">=") &&
            !(home?.yearBuilt >= yearBuiltNum)
          ) {
            include = false;
          }
          if (
            filters.yearBuilt.includes("<=") &&
            !(home?.yearBuilt <= yearBuiltNum)
          ) {
            include = false;
          }
        }

        if (filters.garageSpaces && home?.garageSpaces !== undefined) {
          const garageSpacesNum = parseInt(
            filters.garageSpaces.replace(/[^0-9]/g, ""),
            10
          );
          if (
            filters.garageSpaces.includes(">") &&
            !(home?.garageSpaces > garageSpacesNum)
          ) {
            include = false;
          }
          if (
            filters.garageSpaces.includes("<") &&
            !(home?.garageSpaces < garageSpacesNum)
          ) {
            include = false;
          }
          if (
            filters.garageSpaces.includes(">=") &&
            !(home?.garageSpaces >= garageSpacesNum)
          ) {
            include = false;
          }
          if (
            filters.garageSpaces.includes("<=") &&
            !(home?.garageSpaces <= garageSpacesNum)
          ) {
            include = false;
          }
        }

        if (filters.lotSize && home?.lotSize !== undefined) {
          const lotSizeNum = parseFloat(
            filters.lotSize.replace(/[^0-9.]/g, "")
          );
          if (filters.lotSize.includes(">") && !(home?.lotSize > lotSizeNum)) {
            include = false;
          }
          if (filters.lotSize.includes("<") && !(home?.lotSize < lotSizeNum)) {
            include = false;
          }
          if (
            filters.lotSize.includes(">=") &&
            !(home?.lotSize >= lotSizeNum)
          ) {
            include = false;
          }
          if (
            filters.lotSize.includes("<=") &&
            !(home?.lotSize <= lotSizeNum)
          ) {
            include = false;
          }
        }

        if (filters.neighborhood && home?.neighborhood !== undefined) {
          if (
            !home?.neighborhood
              .toLowerCase()
              .includes(filters.neighborhood.toLowerCase())
          ) {
            include = false;
          }
        }

        if (filters.zipCode && home?.zipCode !== undefined) {
          if (!home?.zipCode.startsWith(filters.zipCode)) {
            include = false;
          }
        }

        return include;
      });
    },
    prepareChartData(data, key) {
      const chartData = {};
      data.forEach((item) => {
        if (chartData[item[key]]) {
          chartData[item[key]] += 1;
        } else {
          chartData[item[key]] = 1;
        }
      });
      return chartData;
    },
    renderChart(config) {
      if (this.chart) {
        this.chart.destroy();
      }
      const ctx = document.getElementById("chart").getContext("2d");
      this.chart = new Chart(ctx, {
        type: config.type,
        data: {
          labels: Object.keys(this.chartData),
          datasets: [
            {
              data: Object.values(this.chartData),
              backgroundColor: config.colors,
            },
          ],
        },
        options: config.options,
      });
    },
  },
  mounted() {
    this.fetchHousingData();
  },
};
</script>

<style scoped>
.chat-container {
  max-width: 600px;
  margin: auto;
  padding: 20px;
  text-align: center;
}
textarea {
  width: 100%;
  height: 100px;
  margin-bottom: 10px;
}
button {
  padding: 10px 20px;
}
canvas {
  max-width: 100%;
  margin-top: 20px;
}
</style>
