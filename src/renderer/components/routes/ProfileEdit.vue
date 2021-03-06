<template>
  <page>
    <template slot="title">
      {{ $t('ProfileEdit.EditProfile') }}
    </template>

    <template slot="body">
      <template v-if="publishing">
        <b-progress type="is-info"></b-progress>
      </template>
      <template v-else>
        <b-field :label="$t('ProfileEdit.Name')">
          <b-input v-model="name"></b-input>
        </b-field>

        <b-field :label="$t('ProfileEdit.AccountType')">
          <b-select v-model="type">
            <option value="0">{{ $t('ProfileEdit.Anon') }}</option>
            <option value="1">{{ $t('ProfileEdit.Person') }}</option>
            <option value="2">{{ $t('ProfileEdit.Project') }}</option>
            <option value="3">{{ $t('ProfileEdit.Organization') }}</option>
            <option value="4">{{ $t('ProfileEdit.Proxy') }}</option>
            <option value="5">{{ $t('ProfileEdit.Parody') }}</option>
            <option value="6">{{ $t('ProfileEdit.Bot') }}</option>
            <option value="7">{{ $t('ProfileEdit.Shill') }}</option>
            <option value="8">{{ $t('ProfileEdit.Test') }}</option>
          </b-select>
        </b-field>

        <b-field :label="$t('ProfileEdit.Location')">
          <b-input v-model="location"></b-input>
        </b-field>

        <b-field :label="$t('ProfileEdit.Bio')">
          <b-input v-model="bio" type="textarea"></b-input>
        </b-field>

        <b-field class="file">
          <b-upload v-model="file">
            <a class="button is-primary">
              <b-icon icon="upload"></b-icon>
              <span>{{ $t('ProfileEdit.ChooseImage') }}</span>
            </a>
          </b-upload>
          <span class="file-name" v-if="file">
            {{ file.name }}
          </span>
        </b-field>

        <button class="button is-primary" @click="publish">{{ $t('ProfileEdit.Publish') }}</button>
      </template>
    </template>
  </page>
</template>

<script lang="ts">
  import Vue from 'vue'
  import Page from '../Page.vue'
  let ProfileMixinProto: any = require('../../../lib/protobuf/ProfileMixin_pb.js')
  let TitleMixinProto: any = require('../../../lib/protobuf/TitleMixin_pb.js')
  let BodyTextMixinProto: any = require('../../../lib/protobuf/BodyTextMixin_pb.js')
  let LanguageMixinProto: any = require('../../../lib/protobuf/LanguageMixin_pb.js')
  import MixItem from '../../../lib/MixItem'
  import Image from '../../../lib/Image'
  import MixContent from '../../../lib/MixContent'
  import setTitle from '../../../lib/setTitle'
  import bs58 from 'bs58'

  export default Vue.extend({
    name: 'profile-edit',
    components: {
      Page,
    },
    data() {
      return {
        publishing: false,
        name: '',
        bio: '',
        location: '',
        type: '',
        file: null,
      }
    },
    async created() {
      if (!this.$activeAccount.get()) {
        return
      }
      let itemId = await this.$activeAccount.get().getProfile()
      let item = await new MixItem(this.$root, itemId).init()
      let revision = await item.latestRevision().load()
      this.name = revision.getTitle()
      setTitle(this.name)
      this.bio = revision.getBodyText()
      let profile = revision.getProfile()
      this.type = profile.type
      this.location = profile.location
    },
    methods: {
      async publish(event: any) {
        this.publishing = true
        try {
          let content = new MixContent(this.$root)

          // Account profile
          let profileMessage = new ProfileMixinProto.ProfileMixin()
          profileMessage.setType(this.type)
          profileMessage.setLocation(this.location)
          content.addMixinPayload('0xbeef2144', profileMessage.serializeBinary())

          // Language
          let languageMessage = new LanguageMixinProto.LanguageMixin()
          languageMessage.setLanguageTag(this.$settings.get('locale'))
          content.addMixinPayload('0x9bc7a0e6', languageMessage.serializeBinary())

          // Title
          let titleMessage = new TitleMixinProto.TitleMixin()
          titleMessage.setTitle(this.name)
          content.addMixinPayload('0x344f4812', titleMessage.serializeBinary())

          // BodyText
          let bodyTextMessage = new BodyTextMixinProto.BodyTextMixin()
          bodyTextMessage.setBodyText(this.bio)
          content.addMixinPayload('0x2d382044', bodyTextMessage.serializeBinary())

          // Image
          if (this.file == null) {
            try {
              let itemId = await this.$activeAccount.get().getProfile()
              let item = await new MixItem(this.$root, itemId).init()
              let revision = await item.latestRevision().load()
              content.addMixinPayload('0x045eee8c', revision.content.getPayloads('0x045eee8c')[0])
            }
            catch (e) {}
          }
          else {
            let image = new Image(this.$root, this.file)
            content.addMixinPayload('0x045eee8c', await image.createMixin())
          }

          let ipfsHash = await content.save()
          let itemId
          try {
            itemId = await this.$activeAccount.get().call(this.$mixClient.accountProfile, 'getProfile')
            await this.$activeAccount.get().sendData(this.$mixClient.itemStoreIpfsSha256, 'createNewRevision', [itemId, ipfsHash], 0, 'Update profile')
          }
          catch(e) {
            let flagsNonce = '0x01' + this.$mixClient.web3.utils.randomHex(31).substr(2)
            itemId = await this.$activeAccount.get().call(this.$mixClient.itemStoreIpfsSha256, 'getNewItemId', [this.$activeAccount.get().contractAddress, flagsNonce])
            await this.$activeAccount.get().sendData(this.$mixClient.itemStoreIpfsSha256, 'create', [flagsNonce, ipfsHash], 0, 'Create profile item')
            await this.$activeAccount.get().sendData(this.$mixClient.accountProfile, 'setProfile', [itemId], 0, 'Set profile item')
          }

          this.$root.$emit('change-active-account', this.$activeAccount.get())
          let encodedItemId = bs58.encode(Buffer.from(this.$mixClient.web3.utils.hexToBytes(itemId.substr(0, 50))))
          this.$router.push({ name: 'item', params: { encodedItemId: encodedItemId }})
        }
        catch (e) {
          this.publishing = false
        }
      }
    }
  })
</script>
