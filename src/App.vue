<template>
  <div class="min-w-[30rem]">
    <nav class="text-[1.1rem] font-bold p-4">코인시세보기</nav>
    <div>
      <table class="table w-full">
        <thead>
          <tr class="text-sm text-gray-500 border-b border-gray-100">
            <th class="py-2 px-4 text-left">한글명</th>
            <th class="py-2 px-4 text-right">현재가(원)</th>
            <th class="py-2 px-4 text-right">전일대비</th>
            <th class="py-2 px-4 text-right">거래대금</th>
          </tr>
        </thead>
        <tbody class="text-gray-900">
          <tr v-for="(coin, key) in coins" :key="key">
            <td class="py-1 px-4 text-left">
              <div class="font-semibold text-gray-700">
                {{ coin.korean_name }}
              </div>
              <div class="text-sm text-gray-500">{{ coin.market }}</div>
            </td>
            <td
              :class="[
                GetColor(coin),
                'py-1 px-4 text-right font-semibold align-top',
              ]"
            >
              {{ GetRatePrefix(coin) }}{{ GetCurrency(coin.trade_price) }}
            </td>
            <td :class="[GetColor(coin), 'py-1 px-4 text-right']">
              <div class="font-semibold">
                {{ GetRatePrefix(coin) }}
                {{ GetChangeRate(coin.change_rate) }}%
              </div>
              <div class="text-sm text-gray-500">
                {{ GetRatePrefix(coin) }}
                {{ GetCurrency(coin.change_price) }}원
              </div>
            </td>
            <td class="py-1 px-4 text-right">
              {{ getVolume(coin.acc_trade_price_24h) }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    <footer class="font-[0.9rem] text-gray-600 p-4 border-t border-gray-300">
      pinako
    </footer>
  </div>
</template>
<script setup>
import { onMounted, ref } from "vue";
import axios from "axios";
import { numToKorean } from "num-to-korean";
import { v4 as uuidv4 } from "uuid";

const coins = ref({});
const markets = ref([]);
onMounted(async () => {
  try {
    const { data: symbols } = await GetSymbols();
    // console.log(symbols);
    symbols.forEach((symbol) => {
      /** `KRW-` 으로 시작하는 마켓만 핕터링 */
      if (symbol.market.indexOf("KRW-") === -1) {
        return;
      }
      markets.value.push(symbol.market);
      coins.value[symbol.market] = symbol;
    });
    const ws = new WebSocket("wss://api.upbit.com/websocket/v1");
    ws.onopen = (e) => {
      ws.send(
        `${JSON.stringify([
          { ticket: uuidv4() },
          { type: "ticker", codes: markets.value },
        ])}`
      );
    };
    ws.onmessage = async (payload) => {
      const ticker = await new Response(payload.data).json();
      // console.log(ticker);
      if (!coins.value[ticker.code]) {
        return;
      }
      if (ticker.change === "FALL") {
        coins.value[ticker.code].trade_price = -ticker.trade_price;
        coins.value[ticker.code].change_rate = -ticker.change_rate;
        coins.value[ticker.code].change_price = -ticker.change_price;
      } else {
        coins.value[ticker.code].trade_price = ticker.trade_price;
        coins.value[ticker.code].change_rate = ticker.change_rate;
        coins.value[ticker.code].change_price = ticker.change_price;
      }
      coins.value[ticker.code].change = ticker.change;
      coins.value[ticker.code].acc_trade_price_24h = ticker.acc_trade_price_24h;
    };
  } catch (err) {}
});

// console.log(coins.value);
function GetSymbols() {
  return axios.get("https://api.upbit.com/v1/market/all");
}

function GetTickers(markets = []) {
  return axios.get("https://api.upbit.com/v1/ticker", {
    params: { markets: markets.join(",") },
  });
}

function GetCurrency(value) {
  return Number(value).toLocaleString();
}

function GetChangeRate(value) {
  return Number(value).toFixed(2);
}

function getVolume(volume) {
  return numToKorean(Math.floor(volume / 100000000) * 100000000, "mixed");
}

function GetRatePrefix(coin) {
  switch (coin.change) {
    case "RISE":
      return "+";
    default:
      return "";
  }
}

function GetColor(coin) {
  switch (coin.change) {
    case "RISE":
      return "text-red-600";
    case "FALL":
      return "text-blue-600";
    default:
      return "text-gray-600";
  }
}
</script>

