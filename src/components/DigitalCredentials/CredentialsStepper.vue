<template>
  <div id="credentials-stepper">
    <!-- Dialogs -->
    <ConfirmCredentialsTermsOfUseDialog
      :dialog="confirmCredentialsTermsOfUseDialog"
      attach="#app"
      @close="goToCredentialsDashboard()"
      @proceed="hideConfirmCredentialsTermsOfUseDialog()"
    />

    <CredentialNotReceivedDialog
      :dialog="credentialNotReceivedDialog"
      attach="#app"
      @close="credentialNotReceivedDialog = false"
      @onResendOffer="resendOffer()"
      @onResetOffer="resetOffer()"
    />

    <!-- Initial Page Load Transition -->
    <v-fade-transition>
      <div
        v-show="showLoadingContainer"
        class="loading-container"
      >
        <div class="loading__content">
          <v-progress-circular
            color="primary"
            :size="50"
            indeterminate
          />
          <div class="loading-msg">
            {{ loadingMessage }}
          </div>
        </div>
      </div>
    </v-fade-transition>

    <v-container>
      <v-row>
        <v-col
          cols="12"
          md="8"
        >
          <v-card class="pa-3">
            <v-card-text>
              <h1>Steps to get your credential:</h1>
              <v-card class="mt-6">
                <v-card-text class="d-flex">
                  <span
                    class="credentials-stepper-number ml-n4 my-n4 px-8
                    font-weight-bold align-self-stretch d-flex rounded-l"
                  >
                    <div class="align-self-center">1</div>
                  </span>
                  <span class="ml-8">
                    Have the
                    <a
                      href="https://www2.gov.bc.ca/gov/content/governments/government-id/bc-wallet"
                      target="_blank"
                    >
                      BC Wallet app<v-icon
                        small
                        color="primary"
                      >mdi-open-in-new</v-icon></a>
                    installed on your mobile phone.
                  </span>
                </v-card-text>
              </v-card>
              <v-card class="mt-2">
                <v-card-text class="d-flex">
                  <span
                    class="credentials-stepper-number ml-n4 my-n4 px-8
                    font-weight-bold align-self-stretch d-flex rounded-l"
                  >
                    <div class="align-self-center">2</div>
                  </span>
                  <span class="ml-8">
                    Scan the QR code with your BC Wallet to get your Digital Business Card and accept it in your BC
                    Wallet app.
                  </span>
                </v-card-text>
              </v-card>
              <v-card class="mt-2">
                <v-card-text class="d-flex">
                  <span
                    class="credentials-stepper-number ml-n4 my-n4 px-8
                    font-weight-bold align-self-stretch d-flex rounded-l"
                  >
                    <div class="align-self-center">3</div>
                  </span>
                  <span class="ml-8">
                    Check your Digital Business Cards on the
                    <router-link to="/digital-credentials">credential dashboard</router-link>.
                  </span>
                </v-card-text>
              </v-card>
            </v-card-text>
          </v-card>
        </v-col>
        <v-col
          cols="12"
          md="4"
        >
          <v-card
            v-if="!credentialConnection?.isActive"
            class="pt-3 px-3"
          >
            <v-card-text class="d-flex flex-column">
              <p class="justify-center text-center font-weight-bold word-break-normal">
                Scan this QR code with your BC Wallet app
              </p>
              <QrcodeVue
                class="d-flex justify-center mt-4"
                render-as="svg"
                size="200"
                :value="credentialConnection?.invitationUrl"
              />
              <p class="mt-8 justify-center text-center word-break-normal">
                QR code isn't scanning?
                <a
                  href="#"
                  @click.prevent="handleGenegerateNewQRCode()"
                >
                  Generate a new QR code
                </a>.
              </p>
            </v-card-text>
          </v-card>
          <v-card
            v-else
            class="pt-3 px-3"
          >
            <v-card-text
              v-if="!issuedCredential?.isIssued"
              class="d-flex flex-column"
            >
              <p class="justify-center text-center font-weight-bold word-break-normal">
                Accept the request in your wallet
              </p>
              <div
                class="d-flex justify-self-center align-self-center justify-center align-center pt-4"
                style="width: 200px; height: 200px"
              >
                <v-progress-circular
                  color="primary"
                  indeterminate
                />
              </div>
              <p class="justify-center text-center word-break-normal pt-8">
                <a
                  href="#"
                  @click.prevent="handleNoCredentialOfferReceived()"
                >
                  I didn't receive anything
                </a>
              </p>
            </v-card-text>
            <v-card-text
              v-else
              class="d-flex flex-column"
            >
              <p class="justify-center text-center font-weight-bold word-break-normal">
                Your Business Card is now ready to use
              </p>
              <div
                class="d-flex justify-self-center align-self-center justify-center align-center"
                style="width: 200px; height: 200px"
              >
                <v-icon
                  color="black"
                  size="xxx-large"
                >
                  mdi-check-circle-outline
                </v-icon>
              </div>
              <p class="justify-center text-center word-break-normal pt-8">
                Go to your
                <router-link to="/digital-credentials">
                  credential dashboard
                </router-link>
                to see details and make changes.
              </p>
            </v-card-text>
          </v-card>
        </v-col>
      </v-row>
    </v-container>
    <CredentialsWebSocket
      @onConnection="handleConnection($event)"
      @onIssuedCredential="handleIssuedCredential($event)"
    />
  </div>
</template>

<script lang="ts">
import { DigitalCredentialTypes, Routes } from '@/enums'
import { LegalServices } from '@/services'
import { useBusinessStore } from '@/stores'
import { Getter } from 'pinia-class'
import { Component, Vue } from 'vue-property-decorator'
import QrcodeVue from 'qrcode.vue'
import { DigitalCredentialIF, WalletConnectionIF } from '@/interfaces'
import CredentialsWebSocket from '@/components/DigitalCredentials/CredentialsWebSocket.vue'
import ConfirmCredentialsTermsOfUseDialog
  from '@/components/DigitalCredentials/dialogs/ConfirmCredentialsTermsofUseDialog.vue'
import CredentialNotReceivedDialog from './dialogs/CredentialNotReceivedDialog.vue'

Component.registerHooks(['beforeRouteEnter'])

// Create a component that extends the Vue class called CredentialsStepper
@Component({
  components: {
    QrcodeVue,
    CredentialsWebSocket,
    ConfirmCredentialsTermsOfUseDialog,
    CredentialNotReceivedDialog
  }
})
export default class CredentialsStepper extends Vue {
  @Getter(useBusinessStore) getIdentifier!: string

  loadingMessage = 'Loading'
  showLoadingContainer = true
  confirmCredentialsTermsOfUseDialog = false
  credentialNotReceivedDialog = false
  credentialTypes = DigitalCredentialTypes
  credentialConnection: WalletConnectionIF = null
  issuedCredential: DigitalCredentialIF = null

  async beforeRouteEnter (to, from, next): Promise<void> {
    next(async (_this) => {
      const { data } = await LegalServices.fetchCredentials(_this.getIdentifier)
      const issuedCredentials = data?.issuedCredentials

      if (issuedCredentials?.length) {
        next({ name: Routes.DIGITAL_CREDENTIALS })
      } else {
        _this.confirmCredentialsTermsOfUseDialog = true
        next()
      }
    })
  }

  async setupConnection (): Promise<void> {
    await this.getCredentialsConnection()
    if (!this.credentialConnection) {
      await this.addCredentialInvitation()
    }
    this.showLoadingContainer = false
  }

  async hideConfirmCredentialsTermsOfUseDialog (): Promise<void> {
    this.confirmCredentialsTermsOfUseDialog = false
    await this.setupConnection()
  }

  async handleGenegerateNewQRCode (): Promise<void> {
    this.showLoadingContainer = true
    await LegalServices.removeCredentialConnection(this.getIdentifier, this.credentialConnection.connectionId)
    await this.setupConnection()
  }

  async handleNoCredentialOfferReceived (): Promise<void> {
    this.credentialNotReceivedDialog = true
  }

  async addCredentialInvitation (): Promise<void> {
    const { data: connection } = await LegalServices.createCredentialInvitation(this.getIdentifier)
    this.credentialConnection = connection || null
  }

  async getCredentialsConnection (): Promise<void> {
    const { data } = await LegalServices.fetchCredentialConnections(this.getIdentifier)
    this.credentialConnection = data?.connections?.[0] || null
  }

  async issueCredential (credentialType: DigitalCredentialTypes): Promise<void> {
    const { data: issuedCredential } = await LegalServices.sendCredentialOffer(this.getIdentifier, credentialType)
    this.issuedCredential = issuedCredential || null
  }

  async handleConnection (connection: WalletConnectionIF) {
    this.credentialConnection = connection
    await this.issueCredential(this.credentialTypes.BUSINESS)
  }

  async handleIssuedCredential (issuedCredential: DigitalCredentialIF) {
    this.issuedCredential = issuedCredential
  }

  goToCredentialsDashboard (): void {
    this.$router.push({ name: Routes.DIGITAL_CREDENTIALS })
  }

  async resendOffer (): Promise<void> {
    this.credentialNotReceivedDialog = false
    this.showLoadingContainer = true
    if (this.issuedCredential) {
      await LegalServices.removeCredential(this.getIdentifier, this.issuedCredential.credentialId)
    }
    await this.issueCredential(this.credentialTypes.BUSINESS)
    this.showLoadingContainer = false
  }

  async resetOffer (): Promise<void> {
    this.credentialNotReceivedDialog = false
    this.showLoadingContainer = true
    if (this.issuedCredential) {
      await LegalServices.removeCredential(this.getIdentifier, this.issuedCredential.credentialId)
    }
    if (this.credentialConnection) {
      await LegalServices.removeCredentialConnection(this.getIdentifier, this.credentialConnection.connectionId)
    }
    await this.setupConnection()
  }
}
</script>

<style lang="scss" scoped>
@import "@/assets/styles/theme.scss";

.credentials-stepper-number {
  color: white;
  background-color: $app-blue;
}

.loading-container {
  z-index: 1;
}
</style>