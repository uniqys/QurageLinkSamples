<template>
  <div class="box">
    <h2>{{ title }}</h2>
    <button
      class="button is-primary"
      @click="$emit('http')"
      :disabled="disabledHttp"
    >
      web3 0.20
    </button>
    <button
      class="button is-primary"
      @click="$emit('ethereum')"
      :disabled="disabledEthereum"
    >
      web3 1.0
    </button>
    <p class="message">{{ message }}</p>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'nuxt-property-decorator'

@Component
export default class SignMessageContent extends Vue {
  @Prop() title!: string
  @Prop() message!: string
  @Prop() disabledHttp?: boolean
  @Prop() disabledEthereum?: boolean
}
</script>
