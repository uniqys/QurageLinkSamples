<template>
  <div class="card block">
    <CardHeader>
      <CardTitle>Sign Message</CardTitle>
      <CardIcon />
    </CardHeader>

    <CardContent>
      <template v-if="isLinked">
        <div class="test-sign-message">
          <SignMessageContent
            title="Net version"
            :message="message.netVersion"
            @http="netVersionHttp()"
            @ethereum="netVersionEthereum()"
          />
          <SignMessageContent
            title="Sign message"
            :message="message.eth"
            @http="ethSignHttp(signMessage)"
            @ethereum="ethSignEthereum(signMessage)"
          />
          <SignMessageContent
            title="Personal message"
            :message="message.personal"
            @http="personalSignHttp(personalSignText)"
            @ethereum="personalSignEthereum(personalSignText)"
          />
          <SignMessageContent
            title="Typed message"
            :message="message.typed"
            @http="typedMessageHttp(typedMessage)"
            @ethereum="typedMessageEthereum(typedMessage)"
          />
          <SignMessageContent
            :title="'EC Recover from Personal message'"
            :message="recoveredMessage"
            @http="ecRecoverHttp(personalSignText, (message.personal).split(': ')[1])"
            @ethereum="ecRecoverEthereum(personalSignText, (message.personal).split(': ')[1])"
          />
        </div>
      </template>
      <template v-else>
        not linked
      </template>
      <TextInfo v-if="isSigning">
        Signing...
      </TextInfo>
    </CardContent>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'nuxt-property-decorator'
import { JsonRPCResponse, ProviderMethod, Callback } from '@uniqys/qurage-link-lib'

import CardTitle from '../atoms/CardTitle.vue'
import CardIcon from '../atoms/CardIcon.vue'
import CardHeader from '../molecules/CardHeader.vue'
import CardContent from '../molecules/CardContent.vue'
import SignMessageContent from '../molecules/SignMessageContent.vue'
import TextInfo from '../atoms/TextInfo.vue'


interface SignedMessage {
  eth: string
  personal: string
  typed: string
  netVersion: string
}

@Component({
  components: {
    TextInfo,
    SignMessageContent,
    CardHeader,
    CardTitle,
    CardIcon,
    CardContent,
  },
})
export default class SignMessageCard extends Vue {
  @Prop() isLinked!: boolean
  isSigning: boolean = false

  readonly signMessage = '0x879a053d4800c6354e76c7985a865d2922c82fb5b3f4577b2fe08b998954f2e0'
  readonly personalSignText = 'Hello, Quragé Link!'
  readonly typedMessage = JSON.stringify({
    types: {
      EIP712Domain: [
        { name: "name", type: "string" },
        { name: "version", type: "string" },
        { name: "chainId", type: "uint256" },
        { name: "verifyingContract", type: "address" },
      ],
      Person: [
        { name: "name", type: "string" },
        { name: "wallet", type: "address" },
      ],
      Mail: [
        { name: "from", type: "Person" },
        { name: "to", type: "Person" },
        { name: "contents", type: "string" },
      ],
    },
    primaryType: "Mail",
    domain: {
      name: "Ether Mail",
      version: "1",
      chainId: 1,
      verifyingContract: "0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC",
    },
    message: {
      from: { name: "Cow", wallet: "0xCD2a3d9F938E13CD947Ec05AbC7FE734Df8DD826" },
      to: { name: "Bob", wallet: "0xbBbBBBBbbBBBbbbBbbBbbbbBBbBbbbbBbBbbBBbB" },
      contents: "Hello, Bob!",
    },
  })

  recoveredMessage = ''
  message: SignedMessage = {
    eth: '',
    personal: '',
    typed: '',
    netVersion: '',
  }

  private get web3VerZero (): any {
    return (window as any).web3
  }

  private get web3VerOne (): any {
    return (window as any).web3['1.0']
  }

  netVersionHttp () {
    const web3 = this.web3VerZero
    try {
      const netVersion = web3.version.network
      this.message.netVersion = netVersion
    } catch (error) {
      console.error(error)
    }
  }

  async netVersionEthereum () {
    const web3 = this.web3VerOne
    try {
      const netVersion = await web3.eth.net.getId()
      this.message.netVersion = netVersion
    } catch (error) {
      console.error(error)
    }
  }

  ethSignHttp (message: string) {
    this.isSigning = true

    const web3 = this.web3VerZero

    const from = web3.eth.defaultAccount.toLowerCase()
    web3.eth.sign(from, message, (err: Error, result: string) => {
      try {
        if (err) {
          return console.error(err)
        }
        this.message.eth = `SIGNED: ${result}`
      } finally {
        this.isSigning = false
      }
    })
  }

  async ethSignEthereum (message: string): Promise<void> {
    await this.signAsyncWithState(async () => {
      const web3 = this.web3VerOne

      const [from] = await this.getAddress(web3)
      await web3.eth.sign(message, from)
        .then((sign: string) => this.message.eth = `SIGNED: ${sign}`)
        .catch((err: Error) => console.error(err))
    })
  }

  async personalSignHttp (text: string): Promise<void> {
    await this.signAsyncWithState(async () => {
      const web3 = this.web3VerZero
      const from = web3.eth.defaultAccount.toLowerCase()
      const callback: Callback<JsonRPCResponse> = (err: Error | null, result?: JsonRPCResponse) => {
        if (err) { return console.error(err) }
        this.message.personal = `PERSONAL SIGNED: ${result!.result}`
      }

      await web3.currentProvider.sendAsync({
        from,
        method: ProviderMethod.PersonalSign,
        params: [from, text],
        jsonrpc: '2.0',
      }, callback)
    })
  }

  async personalSignEthereum (text: string): Promise<void> {
    await this.signAsyncWithState(async () => {
      const web3 = this.web3VerOne

      const [from] = await web3.eth.getAccounts()
      await web3.currentProvider.send(ProviderMethod.PersonalSign, [from, text])
        .then((result: string) => this.message.personal = `PERSONAL SIGNED: ${result}`)
        .catch((err: Error) => console.error(err))
    })
  }

  async typedMessageHttp (text: string): Promise<void> {
    await this.signAsyncWithState(async () => {
      const web3 = this.web3VerZero

      const from = web3.eth.defaultAccount.toLowerCase()
      const callback: Callback<JsonRPCResponse> = (err: Error | null, result?: JsonRPCResponse) => {
        if (err) { return console.error(err) }
        this.message.typed = `SIGNED TYPED MESSAGE: ${result!.result}`
      }
      await web3.currentProvider.sendAsync({
        from,
        method: ProviderMethod.EthSignTypedDataV3,
        params: [from, text],
        jsonrpc: '2.0'
      }, callback)
    })
  }

  async typedMessageEthereum (text: string): Promise<void> {
    await this.signAsyncWithState(async () => {
      const web3 = this.web3VerOne

      const [from] = await web3.eth.getAccounts()
      await web3.currentProvider.send(ProviderMethod.EthSignTypedDataV3, [from, text])
        .then((result: string) => this.message.typed = `SIGNED TYPED MESSAGE: ${result}`)
        .catch((err: Error) => console.error(err))
    })
  }

  async ecRecoverHttp (message: string, signature: string): Promise<void> {
    await this.signAsyncWithState(async () => {
      const web3 = this.web3VerZero

      const from = web3.eth.defaultAccount.toLowerCase()
      const callback: Callback<JsonRPCResponse> = (err: Error | null, result?: JsonRPCResponse) => {
        if (err) {
          return console.error(err)
        }

        const recovered = result!.result
        this.recoveredMessage = `RECOVERED ADDRESS: ${recovered}`
      }

      await web3.currentProvider.sendAsync({
        from,
        method: ProviderMethod.PersonalECRecover,
        params: [message, signature],
        jsonrpc: '2.0',
      }, callback)
    })
  }

  async ecRecoverEthereum (message: string, signature: string): Promise<void> {
    await this.signAsyncWithState(async () => {
      const web3 = this.web3VerOne

      const [from] = await web3.eth.getAccounts()
      const callback: Callback<JsonRPCResponse> = (err: Error | null, result?: JsonRPCResponse) => {
        if (err) {
          return console.error(err)
        }

        const recovered = result!.result;
        this.recoveredMessage = `RECOVERED ADDRESS: ${recovered}`
      }

      await web3.currentProvider.send({
        from,
        method: ProviderMethod.PersonalECRecover,
        params: [message, signature],
        jsonrpc: '2.0',
      }, callback)
    })
  }

  private async signAsyncWithState (signer: () => Promise<void>): Promise<void> {
    this.isSigning = true
    try {
      await signer()
    } finally {
      this.isSigning = false
    }
  }

  private async getAddress (web3: any): Promise<string> {
    return web3.isMetaMask
      ? web3.selectedAddress
      : web3.eth.getAccounts()
  }
}
</script>
