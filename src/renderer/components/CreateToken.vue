<template>
  <page>
    <template slot="title">
      {{ $t('CreateToken.CreateToken') }}
    </template>

    <template slot="body">
      <b-field :label="$t('CreateToken.Symbol')">
        <b-input v-model="symbol"></b-input>
      </b-field>
      <b-field :label="$t('CreateToken.Name')">
        <b-input v-model="name"></b-input>
      </b-field>
      <b-field :label="$t('CreateToken.Description')">
        <b-input v-model="description" type="textarea"></b-input>
      </b-field>
      <b-field :label="$t('CreateToken.InitialBalance')">
        <b-input v-model="initialBalance"></b-input>
      </b-field>
      <b-field :label="$t('CreateToken.DailyPayout')">
        <b-input v-model="dailyPayout"></b-input>
      </b-field>

      <button class="button" @click="chooseFile">{{ $t('CreateToken.ChooseImage') }}</button>
      <button class="button is-primary" @click="create">{{ $t('CreateToken.Create') }}</button>
    </template>
  </page>
</template>

<script>
  import Page from './Page.vue'
  import LanguageMixinProto from '../../lib/protobuf/LanguageMixin_pb.js'
  import TitleMixinProto from '../../lib/protobuf/TitleMixin_pb.js'
  import BodyTextMixinProto from '../../lib/protobuf/BodyTextMixin_pb.js'
  import MixContent from '../../lib/MixContent.js'
  import Image from '../../lib/Image.js'
  import ethTx from 'ethereumjs-tx'
  import path from 'path'
  import fs from 'fs'
  import setTitle from '../../lib/setTitle.js'

  export default {
    name: 'create-token',
    components: {
      Page,
    },
    data() {
      return {
        symbol: '',
        name: '',
        description: '',
        initialBalance: '',
        dailyPayout: '',
      }
    },
    created() {
      setTitle(this.$t('CreateToken.CreateToken'))
    },
    methods: {
      chooseFile(event) {
        let {dialog} = require('electron').remote
        dialog.showOpenDialog({
          title: this.$t('CreateToken.ChooseImage'),
          filters: [{name: 'Images', extensions: ['webp', 'jpg', 'jpeg', 'png', 'gif', 'tiff', 'svg', 'svgz', 'ppm']}],
        }, (fileNames) => {
          this.filepath = fileNames[0]
        })
      },
      async create(event) {
        let flagsNonce = '0x03' + this.$mixClient.web3.utils.randomHex(31).substr(2)
        let itemId = await this.$activeAccount.get().call(this.$mixClient.itemStoreIpfsSha256, 'getNewItemId', [this.$activeAccount.get().contractAddress, flagsNonce])

        let content = new MixContent(this.$root)

        // Token
        content.addMixinPayload(0x9fbbfaad)

        // Image
        let image = new Image(this.$root, this.filepath)
        content.addMixinPayload(0x045eee8c, await image.createMixin())

        // Language
        let languageMessage = new LanguageMixinProto.LanguageMixin()
        languageMessage.setLanguageTag(this.$settings.get('locale'))
        content.addMixinPayload(0x9bc7a0e6, languageMessage.serializeBinary())

        // Title
        let titleMessage = new TitleMixinProto.TitleMixin()
        titleMessage.setTitle(this.name)
        content.addMixinPayload(0x344f4812, titleMessage.serializeBinary())

        // Body text
        let bodyTextMessage = new BodyTextMixinProto.BodyTextMixin()
        bodyTextMessage.setBodyText(this.description)
        content.addMixinPayload(0x2d382044, bodyTextMessage.serializeBinary())

        let ipfsHash = await content.save()

        await this.$activeAccount.get().sendData(this.$mixClient.itemStoreIpfsSha256, 'create', [flagsNonce, ipfsHash], 0, 'Create token item')
        let tokenContract = await this.$activeAccount.get().deployToken(this.symbol, this.name, itemId, this.$mixClient.web3.utils.toWei(this.initialBalance), this.$mixClient.web3.utils.toWei(this.dailyPayout))
        await this.$activeAccount.get().sendData(this.$mixClient.uniswapFactory, 'createExchange', [tokenContract], 0, 'Deploy Uniswap exchange', 1000000)
        this.$router.push({ name: 'item', params: { itemId: itemId }})
      }
    },
  }
</script>
