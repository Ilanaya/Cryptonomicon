<template>
  <body>
    <div class="container mx-auto flex flex-col items-center bg-gray-100 p-4">
      <!-- <div
        class="fixed w-100 h-100 opacity-80 bg-purple-800 inset-0 z-50 flex items-center justify-center"
      >
        <svg
          class="animate-spin -ml-1 mr-3 h-12 w-12 text-white"
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
        >
          <circle
            class="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            stroke-width="4"
          ></circle>
          <path
            class="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
          ></path>
        </svg>
      </div> -->
      <div class="container">
        <add-ticker
          @input="invalidTicker = false"
          @add-ticker="handleAddTicker"
          :listOfCoinsSymbols="listOfCoinsSymbols"
          :invalidTicker="invalidTicker"
        />
        <hr class="w-full border-t border-gray-600 my-4" />

        <app-filter :tickers="tickers" @paginated-tickers="setTickersToShow" />

        <tickers-list
          @select-ticker="handleSelect"
          @delete-ticker="handleDeleteTicker"
          :tickersToShow="tickersToShow"
          :selectedTicker="selectedTicker"
        />

        <app-graph
          @close-graph="selectedTicker = null"
          @max-graph-elements="fixGraph"
          :selectedTicker="selectedTicker"
          :graph="graph"
        />
      </div>
    </div>
  </body>
</template>

<script>
import { loadAllCoins, subscribeToTicker, unsubscribeFromTicker } from "./api";
import AddTicker from "./components/AddTicker.vue";
import AppGraph from "./components/AppGraph.vue";
import AppFilter from "./components/AppFilter.vue";
import TickersList from "./components/TickersList.vue";
export default {
  name: "App",
  components: { AddTicker, AppGraph, AppFilter, TickersList },
  data() {
    return {
      tickers: [],
      graph: [],
      tickersToShow: [],
      listOfCoinsSymbols: [],
      selectedTicker: null,
      invalidTicker: false,
    };
  },

  created() {
    const windowData = Object.fromEntries(
      new URL(window.location).searchParams.entries()
    );
    const VALID_KEYS = ["filter", "page"];
    VALID_KEYS.forEach((key) => {
      if (windowData[key]) {
        this[key] = windowData[key];
      }
    });

    const tickersData = localStorage.getItem("cryptos-list");

    if (tickersData) {
      this.tickers = JSON.parse(tickersData);
      this.tickers.forEach((ticker) => {
        subscribeToTicker(ticker.name, ticker.currency, (newPrice) => {
          this.updateTicker(ticker.name, newPrice);
        });
      });
    }
    loadAllCoins()
      .then((cryptosDescriptions) => {
        this.listOfCoinsSymbols = Object.values(cryptosDescriptions).map(
          (cryptos) => {
            return cryptos.Symbol;
          }
        );
      })
      .catch((e) => {
        console.error("Error loading coins", e);
      });
  },

  methods: {
    handleAddTicker(ticker) {
      const currentTicker = {
        name: ticker,
        price: "-",
        currency: "USD",
      };
      this.validateExistingTicker(currentTicker)
        ? null
        : (this.tickers = [...this.tickers, currentTicker]);

      subscribeToTicker(
        currentTicker.name,
        currentTicker.currency,
        (newPrice) => {
          this.updateTicker(currentTicker.name, newPrice);
        }
      );
    },

    validateExistingTicker(currentTicker) {
      const tickerAlreadyExists = true;
      const duplicatedTicker = this.tickers.find(
        (existingTicker) =>
          currentTicker.name.toLowerCase() === existingTicker.name.toLowerCase()
      );
      if (duplicatedTicker) {
        this.invalidTicker = true;
        return tickerAlreadyExists;
      }
      return !tickerAlreadyExists;
    },

    setTickersToShow(paginatedTickers) {
      this.tickersToShow = paginatedTickers;
    },
    handleSelect(ticker) {
      this.selectedTicker = ticker;
    },

    updateTicker(tickerName, price) {
      this.tickers
        .filter((t) => t.name === tickerName)
        .forEach((t) => {
          if (t === this.selectedTicker) {
            this.graph.push(price);
          }

          t.price = price;
        });
    },
    fixGraph(maxGraphElements) {
      while (this.graph.length > maxGraphElements) {
        this.graph = this.graph.slice(1);
      }
    },

    handleDeleteTicker(tickerToDelete) {
      this.tickers = this.tickers.filter((t) => t !== tickerToDelete);
      if (this.selectedTicker === tickerToDelete) {
        this.selectedTicker = null;
      }
      unsubscribeFromTicker(tickerToDelete.name);
    },
  },
  watch: {
    selectedTicker() {
      this.graph = [];
    },

    tickers() {
      localStorage.setItem("cryptos-list", JSON.stringify(this.tickers));
    },
  },
};
</script>
