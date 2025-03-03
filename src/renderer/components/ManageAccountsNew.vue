<template>
  <page>
    <template slot="title">
      {{ $t('ManageAccountsNew.CreateAccount') }}
    </template>

    <template slot="messages">
      <b-message type="is-warning">
        <p>{{ $t('ManageAccountsNew.Warning1') }}</p>
        <p>{{ $t('ManageAccountsNew.Warning2') }}</p>
        <p>{{ $t('ManageAccountsNew.Warning3') }}</p>
      </b-message>
      <b-message type="is-info">
        <p>{{ $t('ManageAccountsNew.Info1') }}</p>
        <p>{{ $t('ManageAccountsNew.Info2') }}</p>
      </b-message>
    </template>

    <template slot="body">
      <b-field :label="$t('ManageAccountsNew.RecoveryPhrase')">
        {{ recoveryPhrase }}
      </b-field>
      <b-field :label="$t('ManageAccountsNew.Password')" :type="passwordType" :message="passwordMessage">
        <b-input type="password" v-model="password" password-reveal></b-input>
      </b-field>
      <b-field :label="$t('ManageAccountsNew.RepeatPassword')" :type="passwordRepeatType" :message="passwordRepeatMessage">
        <b-input type="password" v-model="passwordRepeat" password-reveal></b-input>
      </b-field>
      <button class="button" @click="create">{{ $t('ManageAccountsNew.Create') }}</button>
    </template>
  </page>
</template>

<script lang="ts">
  import crypto from 'crypto'
  import * as bip32 from 'bip32'
  import { BIP32Interface } from 'bip32'
  import * as bip39  from 'bip39'
  import keythereum from 'keythereum'
  import Page from './Page.vue'
  import MixAccount from '../../lib/MixAccount.js'
  import setTitle from '../../lib/setTitle.js'

  export default {
    name: 'manage-accounts-new',
    components: {
      Page,
    },
    data() {
      return {
        recoveryPhrase: '',
        password: '',
        passwordType: '',
        passwordMessage: '',
        passwordRepeat: '',
        passwordRepeatType: '',
        passwordRepeatMessage: '',
      }
    },
    methods: {
      async create(event) {
        // Password is required.
        if (this.password == '') {
          this.passwordType = 'is-danger'
          this.passwordMessage = this.$t('ManageAccountsNew.PasswordIsRequired')
          return
        }
        else {
          this.passwordType = ''
          this.passwordMessage = ''
        }
        // Check passwords match.
        if (this.password != this.passwordRepeat) {
          this.passwordRepeatType = 'is-danger'
          this.passwordRepeatMessage = this.$t('ManageAccountsNew.PasswordsDoNotMatch')
          return
        }
        else {
          this.passwordRepeatType = ''
          this.passwordRepeatMessage = ''
        }
        // Calculate private key and controller address.
        let node: BIP32Interface = bip32.fromSeed(await bip39.mnemonicToSeed(this.recoveryPhrase))
        let privateKey: Buffer = Buffer.from(node.derivePath("m/44'/76'/0'/0/0").privateKey)
        let controllerAddress: String = keythereum.privateKeyToAddress(privateKey)
        // Encrypt private key.
        let salt: Buffer = crypto.randomBytes(32)
        let iv: Buffer = crypto.randomBytes(16)
        let keyObject: Object = keythereum.dump(this.password, privateKey, salt, iv)
        // Store account in database.
        await this.$db.batch()
        .put('/account/controllerAddress/' + controllerAddress, controllerAddress)
        .put('/account/controller/' + controllerAddress + '/keyObject', JSON.stringify(keyObject))
        .write()
        // Unlock account and goto activation page.
        let account: MixAccount = await new MixAccount(this, controllerAddress).init()
        account.unlock(this.password)
        this.$router.push({ name: 'manage-account-activate', params: { controllerAddress: controllerAddress } })
      },
    },
    async created() {
      setTitle(this.$t('ManageAccountsNew.CreateAccount'))
      this.recoveryPhrase = bip39.generateMnemonic()
    },
  }
</script>
