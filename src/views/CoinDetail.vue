<template>
  <div class="flex-col">
    <div class="flex justify-center">
      <bounce-loader :loading="isLoading" :color="'#68d391'" :size="100" />
    </div>
    <template v-if="!isLoading">
      <div class="flex flex-col sm:flex-row justify-around items-center">
        <div class="flex flex-col items-center">
          <img
            :src="`https://static.coincap.io/assets/icons/${asset.symbol.toLowerCase()}@2x.png`"
            class="w-20 h-20 mr-5"
            :alt="asset.name"
          />
          <h1 class="text-5xl">
            {{ asset.name }}
            <small class="sm:mr-2 text-gray-500">{{ asset.symbol }}</small>
          </h1>
        </div>

        <div class="my-10 flex flex-col">
          <ul>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Ranking</b>
              <span># {{ asset.rank }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio actual</b>
              <span> {{ asset.priceUsd | dollar }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio más bajo</b>
              <span>{{ min | dollar }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio más alto</b>
              <span>{{ max | dollar }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio Promedio</b>
              <span>{{ avg | dollar }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Variación 24hs</b>
              <span>{{ asset.changePercent24Hr | percent }}</span>
            </li>
          </ul>
        </div>

        <div class="my-10 sm:mt-0 flex flex-col justify-center text-center">
          <button
            @click="toggleConverted"
            class="
              bg-green-500
              hover:bg-green-700
              text-white
              font-bold
              py-2
              px-4
              rounded
            "
          >
            {{fromUSD ? `USD a ${asset.symbol}` : `${asset.symbol} a USD`}}
          </button>

          <div class="flex flex-row my-5">
            <label class="w-full" for="convertValue">
              <input
                v-model="convertValue"
                id="convertValue"
                type="number"
                class="
                  text-center
                  bg-white
                  focus:outline-none
                  focus:shadow-outline
                  border border-gray-300
                  rounded-lg
                  py-2
                  px-4
                  block
                  w-full
                  appearance-none
                  leading-normal
                "
              />
            </label>
          </div>

          <span class="text-xl"> {{ convertResult }} {{ fromUSD ? asset.symbol : 'USD'}}</span>
        </div>
      </div>

      <line-chart
        class="my-10"
        :colors="['orange']"
        :min="min"
        :max="max"
        :data="history.map((h) => [h.date, parseFloat(h.priceUsd).toFixed(2)])"
      />

      <h3 class="text-xl my-10">Mejores Ofertas de Cambio</h3>
      <table>
        <tr
          v-for="m in markets"
          :key="`${m.exchageId}-${m.priceUsd}`"
          class="border-b"
        >
          <td>
            <b>{{ m.exchangeId }}</b>
          </td>
          <td>{{ m.priceUsd | dollar }}</td>
          <td>{{ m.baseSymbol }} / {{ m.quoteSymbol }}</td>
          <td>
            <px-button
              :is-loading="m.isLoading || false"
              @custom-click="getWebsite(m)"
              v-if="!m.url"
            >
              <slot> Obtener Link </slot>
            </px-button>
            <a v-else class="hover:underline text-green-600" target="_blanck">
              {{ m.url }}
            </a>
          </td>
        </tr>
      </table>
    </template>
  </div>
</template>

<script>
import PxButton from '@/components/PxButton'
import api from '@/api'

export default {
  name: 'CoinDetail',

  components: { PxButton },

  data() {
    return {
      isLoading: false,
      asset: {},
      history: [],
      markets: [],
      fromUSD: true,
      convertValue: null,
    }
  },

  computed: {
      convertResult() {
          if(!this.convertValue) {
              return 0
          }

          const result = this.fromUSD ? this.convertValue / this.asset.priceUsd : this.convertValue * this.asset.priceUsd

          return result.toFixed(4)
      },


    min() {
      return Math.min(
        ...this.history.map((h) => parseFloat(h.priceUsd).toFixed(2))
      )
    },

    max() {
      return Math.max(
        ...this.history.map((h) => parseFloat(h.priceUsd).toFixed(2))
      )
    },

    avg() {
      return (
        this.history.reduce((a, b) => a + parseFloat(b.priceUsd), 0) /
        this.history.length
      )
    },
  },

  created() {
    this.getCoin()
  },

  watch: {
      $route () {
          this.getCoin()
      }
  },

  methods: {
      toggleConverted(){
          this.fromUSD = !this.fromUSD
      },

    getWebsite(exchange) {
      this.$set(exchange, 'isLoading', true)
      return api
        .getExchange(exchange.exchangeId)
        .then((res) => {
          // se usa para evitar el problema de reactividad para poder trackear la url q se a;ade al objeto
          this.$set(exchange, 'url', res.exchangeUrl)
        })
        .finally(() => {
          this.$set(exchange, 'isLoading', false)
        })
    },

    getCoin() {
      //$route represetna la ruta  y todos los valores de la ruta q llamo  :id // y param nos dice q parametro queremos q esid
      const id = this.$route.params.id
      this.isLoading = true
      // aqui se llena el array con toda la informacion solicitada del api de coincap
      // api.getAssetsCripto(id).then((asset) => (this.asset = asset))
      //Esto vainas loca q no entendi pero sirve para calcular min y max and average
      Promise.all([
        api.getAssetsCripto(id),
        api.getAssetsHistory(id),
        api.getMarkets(id),
      ])
        .then(([asset, history, markets]) => {
          this.asset = asset
          this.history = history
          this.markets = markets
        })
        .finally(() => (this.isLoading = false))
    },
  },
}
</script>

<style scoped>
td{
    padding: 10px;
    text-align: center;
}
</style>
