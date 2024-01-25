<template>
  <v-container v-if="showProp">
    <UIExtensionSlot
      name="checkout_page"
      :invoice="invoice"
      :store="store"
      :checkout-page="checkoutPage"
      @closedialog="$emit('closedialog')"
      @update:invoice="$emit('update:invoice', $event)"
    >
      <close-button
        :show="$vuetify.breakpoint.mobile && !checkoutPage"
        style="background-color: #ffffff"
        @closedialog="$emit('closedialog')"
      />
      <v-card v-if="noTabs" flat>
        <v-card-title class="justify-center"> 無資料 </v-card-title>
        <v-card-text class="d-flex justify-center">
          沒有可用的付款方式<br />
          這可能意味著在創建發票時出現了錯誤。<br />
          請要求客服人員確認相關紀錄。      
        </v-card-text>
      </v-card>
      <div v-else>
        <div v-if="checkoutPage">
          <UIExtensionSlot
            name="checkout_header"
            :logo-url="logoURL"
            :expiration-percentage="expirationPercentage"
            :expiring-soon="expiringSoon"
            :main-progress-color="mainProgressColor"
            :background-progress-color="backgroundProgressColor"
            :timer-text="timerText"
            @closedialog="$emit('closedialog')"
          >
            <v-img contain max-height="40" :src="logoURL" class="mb-2">
              <close-button @closedialog="$emit('closedialog')" />
            </v-img>
            <v-progress-linear
              :value="expirationPercentage"
              height="25"
              :color="mainProgressColor"
              :background-color="backgroundProgressColor"
              class="mx-0"
            >
              <v-progress-circular
                indeterminate
                size="18"
                color="white"
                width="3"
                class="ml-2"
              />
              <p class="my-auto px-3 white--text text-subtitle-2">
                {{
                  expiringSoon
                    ? "發票即將過期..."
                    : "等待付款中..."
                }}
              </p>
              <v-spacer />
              <p class="white--text my-auto text-subtitle-2 px-3">
                {{ timerText }}
              </p>
            </v-progress-linear>
          </UIExtensionSlot>
        </div>
        <v-list class="py-0" dense>
          <UIExtensionSlot
            name="checkout_payment_method_selection"
            :invoice="invoice"
            :itemv="itemv"
            :currency-select-class="currencySelectClass"
            :selected-currency="selectedCurrency"
            @update:selectedCurrency="selectedCurrency = $event"
          >
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title>Pay with</v-list-item-title>
              </v-list-item-content>
              <v-list-item-action>
                <v-menu v-model="showMenu" offset-y style="max-width: 600px">
                  <template #activator="{ on, attrs }">
                    <v-list-item-title
                      class="text-subtitle-1 font-weight-light"
                      :class="currencySelectClass"
                      v-bind="attrs"
                      v-on="on"
                      >{{ itemv.name }}
                      <v-icon v-if="invoice.payments.length > 1" small
                        >mdi-chevron-down</v-icon
                      ></v-list-item-title
                    >
                  </template>
                  <v-list class="overflow-y-auto" style="max-height: 100vh">
                    <v-list-item-group v-model="selectedCurrency" mandatory>
                      <v-list-item
                        v-for="(item, index) in invoice.payments"
                        :key="index"
                      >
                        <v-list-item-title>{{ item.name }}</v-list-item-title>
                      </v-list-item>
                    </v-list-item-group>
                  </v-list>
                </v-menu>
              </v-list-item-action>
            </v-list-item>
          </UIExtensionSlot>
          <v-divider />
          <UIExtensionSlot
            name="checkout_invoice_details"
            :store="store"
            :itemv="itemv"
            :invoice="invoice"
          >
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title>{{ store.name }}</v-list-item-title>
              </v-list-item-content>
              <v-list-item-action>
                <v-list-item-title
                  v-if="!zeroAmountInvoice"
                  class="text-subtitle-1 font-weight-regular align-right"
                  >{{ itemv.amount }}
                  {{ itemv.symbol.toUpperCase() }}</v-list-item-title
                >
                <v-list-item-title :class="rateClass">
                  1 {{ itemv.symbol.toUpperCase() }} =
                  {{ itemv.rate_str }}
                </v-list-item-title>
              </v-list-item-action>
            </v-list-item>
            <v-row v-if="!zeroAmountInvoice" class="mt-n5" no-gutters>
              <v-btn
                icon
                x-small
                class="justify-center mx-auto"
                @click="showDetails = !showDetails"
              >
                <v-icon>{{
                  showDetails ? "mdi-arrow-up" : "mdi-arrow-down"
                }}</v-icon>
              </v-btn>
            </v-row>
            <v-divider v-if="showDetails" />
            <div v-if="showDetails">
              <v-list-item>
                <v-list-item-content>
                  <v-list-item-title>訂單總金額</v-list-item-title>
                </v-list-item-content>
                <v-list-item-action>
                  <v-list-item-title
                    class="text-subtitle-1 font-weight-regular align-right"
                  >
                    {{ networkFeeIncluded ? expectedAmount : itemv.amount }}
                    {{ itemv.symbol.toUpperCase() }}
                  </v-list-item-title>
                  <v-list-item-subtitle>
                    <span
                      v-if="
                        parseFloat(currentPrice) < parseFloat(invoice.price)
                      "
                    >
                      <strike>{{ invoice.price }}</strike>
                      <span class="font-weight-black">
                        {{ currentPrice }}
                      </span>
                    </span>
                    <span v-else>
                      {{ invoice.price }}
                    </span>
                    {{ invoice.currency }}
                  </v-list-item-subtitle>
                </v-list-item-action>
              </v-list-item>
              <v-list-item>
                <v-list-item-content>
                  <v-list-item-title>已經付款</v-list-item-title>
                </v-list-item-content>
                <v-list-item-action>
                  <v-list-item-title
                    class="text-subtitle-1 font-weight-regular align-right"
                  >
                    -{{
                      itemv.name === invoice.paid_currency
                        ? invoice.sent_amount
                        : $utils.decimalStr(0, itemv.divisibility)
                    }}
                    {{ itemv.symbol.toUpperCase() }}
                  </v-list-item-title>
                </v-list-item-action>
              </v-list-item>
              <v-list-item v-if="networkFeeIncluded">
                <v-list-item-content>
                  <v-list-item-title>網路費用</v-list-item-title>
                </v-list-item-content>
                <v-list-item-action>
                  <v-list-item-title
                    class="text-subtitle-1 font-weight-regular align-right"
                  >
                    +{{
                      $utils.decimalStr(
                        parseFloat(itemv.amount) - parseFloat(expectedAmount),
                        itemv.divisibility
                      )
                    }}
                    {{ itemv.symbol.toUpperCase() }}
                  </v-list-item-title>
                </v-list-item-action>
              </v-list-item>
              <v-list-item>
                <v-list-item-content>
                  <v-list-item-title>Due</v-list-item-title>
                </v-list-item-content>
                <v-list-item-action>
                  <v-list-item-title
                    class="text-subtitle-1 font-weight-regular align-right"
                  >
                    {{
                      itemv.name === invoice.paid_currency
                        ? $utils.decimalStr(
                            Math.max(0, itemv.amount - invoice.sent_amount),
                            itemv.divisibility
                          )
                        : itemv.amount
                    }}
                    {{ itemv.symbol.toUpperCase() }}
                  </v-list-item-title>
                </v-list-item-action>
              </v-list-item>
            </div>
          </UIExtensionSlot>
          <v-divider />
          <v-list-item v-if="!needEmail && !needAddress" class="ma-0 pa-0">
            <UIExtensionSlot
              name="checkout_main"
              :checkout-page="checkoutPage"
              :itemv="itemv"
              :update-payment-details="updatePaymentDetails"
              :address-updating="addressUpdating"
              :payment-address-errors="paymentAddressErrors"
              :abi-cache="abiCache"
              :invoice="invoice"
              :store="store"
              :input-payment-address="inputPaymentAddress"
              @update:paymentSelectionFormRef="paymentSelectionFormRef = $event"
              @update:tabsRef="tabsRef = $event"
              @update:inputPaymentAddress="inputPaymentAddress = $event"
            >
              <v-list-item
                v-if="
                  checkoutPage &&
                  (isEthPaymentMethod || itemv.currency === 'trx') &&
                  !itemv.user_address
                "
                class="ma-0 pa-0"
              >
                <v-list-item-content class="ma-0 pa-0">
                  <v-form
                    ref="paymentSelectionForm"
                    @submit.prevent="updatePaymentDetails()"
                  >
                    <v-card flat>
                      <v-card-text>
                        <p class="d-flex justify-center text-h5">
                          選擇支付方式
                        </p>
                        <p class="d-flex justify-center">
                          輸入您將發送資金的錢包地址
                        </p>
                        <v-tooltip bottom class="ma-0 pa-0">
                          <template #activator="{ on, attrs }">
                            <p
                              class="d-flex justify-center blue--text"
                              v-bind="attrs"
                              style="
                                text-decoration: dashed underline !important;
                              "
                              v-on="on"
                            >
                              為什麼我們要詢問您的錢包地址？
                            </p>
                          </template>
                          <p class="ma-0 pa-0">
                            我們不接受來自交易所錢包的資金，所以您需要先將您的資金轉到您自己的錢包。<br />
                            詢問您的錢包地址有助於防止粗心的用戶從交易所發送資金給我們，這往往會導致支付失敗或過期。<br />
                            只有從您指定的錢包地址發送的資金才會被計入您的付款中。
                          </p>
                        </v-tooltip>
                        <v-text-field
                          v-model="inputPaymentAddress"
                          label="您的錢包地址"
                          :error-messages="paymentAddressErrors"
                          :rules="[rules.required]"
                        />
                      </v-card-text>
                      <v-container>
                        <v-row justify="center" class="py-1">
                          <v-btn
                            color="primary"
                            type="submit"
                            :loading="addressUpdating"
                            >Continue with address</v-btn
                          >
                        </v-row>
                        <v-row justify="center" class="py-1">
                          <metamask-button
                            v-if="$device.isDesktop && isEthPaymentMethod"
                            :abi="abiCache"
                            :update-address="updatePaymentDetails"
                            :method="itemv"
                          />
                        </v-row>
                        <v-row justify="center" class="py-1">
                          <wallet-connect-button
                            v-if="$device.isDesktop && isEthPaymentMethod"
                            :method="itemv"
                            :update-address="updatePaymentDetails"
                            :abi="abiCache"
                          />
                        </v-row>
                      </v-container>
                    </v-card>
                  </v-form>
                </v-list-item-content>
              </v-list-item>
              <v-list-item-content class="ma-0 pa-0">
                <v-card class="ma-0 pa-0">
                  <v-card-text class="ma-0 pa-0">
                    <v-container class="ma-0 pa-0">
                      <v-tabs
                        ref="tabs"
                        v-model="selectedAction"
                        centered
                        grow
                        :active-class="checkoutPage ? 'activeTab' : null"
                        :color="checkoutPage ? 'success' : null"
                      >
                        <v-tab>Scan</v-tab>
                        <v-tab>Copy</v-tab>
                      </v-tabs>
                      <v-divider />
                      <v-tabs-items
                        v-model="selectedAction"
                        class="payment-box"
                      >
                        <UIExtensionSlot
                          name="checkout_payment"
                          :itemv="itemv"
                          :store="store"
                          :update-payment-details="updatePaymentDetails"
                          :abi-cache="abiCache"
                          :checkout-page="checkoutPage"
                        >
                          <v-tab-item>
                            <v-container fill-height>
                              <v-row align="center" justify="center">
                                <v-tabs
                                  v-if="itemv.lightning"
                                  v-model="selectedToCopy"
                                  centered
                                  :active-class="
                                    checkoutPage ? 'activeTab' : null
                                  "
                                  :color="checkoutPage ? 'success' : null"
                                >
                                  <v-tab>Invoice</v-tab>
                                  <v-tab>Node Info</v-tab>
                                </v-tabs>
                                <UIExtensionSlot
                                  name="checkout_payment_qr"
                                  :invoice="invoice"
                                  :method="itemv"
                                  :qr-value="qrValue"
                                >
                                  <qrcode
                                    :options="{ width: 240 }"
                                    :value="qrValue"
                                    tag="v-img"
                                    class="d-flex justify-center"
                                  />
                                </UIExtensionSlot>
                              </v-row>
                              <v-row justify="center">
                                <metamask-button
                                  v-if="$device.isDesktop && isEthPaymentMethod"
                                  :abi="abiCache"
                                  :update-address="updatePaymentDetails"
                                  :method="itemv"
                                />
                              </v-row>
                              <v-row justify="center">
                                <UIExtensionSlot
                                  name="checkout_payment_open"
                                  :method="itemv"
                                  :update-address="updatePaymentDetails"
                                  :abi="abiCache"
                                >
                                  <wallet-connect-button
                                    v-if="
                                      $device.isDesktop && isEthPaymentMethod
                                    "
                                    :method="itemv"
                                    :update-address="updatePaymentDetails"
                                    :abi="abiCache"
                                  />
                                  <v-btn
                                    v-else
                                    color="primary"
                                    @click="openInWallet(paymentURL)"
                                    >在錢包中打開</v-btn
                                  >
                                </UIExtensionSlot>
                              </v-row>
                              <v-row v-if="showRecommendedFee" justify="center">
                                Recommended fee:
                                {{ itemv.recommended_fee }} sat/byte
                              </v-row>
                              <v-row v-if="itemv.hint" justify="center">
                                {{ itemv.hint }}
                              </v-row>
                            </v-container>
                          </v-tab-item>
                          <v-tab-item>
                            <v-card flat class="pa-0 ma-0">
                              <v-card-text>
                                <div v-if="!zeroAmountInvoice">
                                  <p class="d-flex justify-center">Amount</p>
                                  <p
                                    class="d-flex justify-center"
                                    :class="
                                      getAmountClass(
                                        itemv.amount.length +
                                          itemv.symbol.length +
                                          1
                                      )
                                    "
                                  >
                                    {{ itemv.amount }}
                                    {{ itemv.symbol.toUpperCase() }}
                                  </p>
                                  <v-divider />
                                </div>
                                <UIExtensionSlot
                                  name="checkout_payment_copy_extra"
                                  :method="itemv"
                                >
                                  <display-field
                                    :title="
                                      itemv.lightning ? 'Invoice' : 'Address'
                                    "
                                    :value="itemv.payment_address"
                                  />
                                  <display-field
                                    v-if="itemv.lightning"
                                    title="節點資訊"
                                    :value="itemv.node_id"
                                  />
                                  <display-field
                                    v-else
                                    title="付款連結"
                                    :value="itemv.payment_url"
                                  />
                                </UIExtensionSlot>
                              </v-card-text>
                            </v-card>
                          </v-tab-item>
                        </UIExtensionSlot>
                      </v-tabs-items>
                    </v-container>
                  </v-card-text>
                  <v-card-actions class="justify-center">
                    <v-btn
                      v-if="!checkoutPage"
                      class="justify-center"
                      color="primary"
                      @click="checkout(invoice.id)"
                    >
                      <v-icon left="left"> mdi-open-in-new </v-icon
                      ><span
                        >Open checkout
                        <v-btn text color="white" @click.stop="showPopup"
                          >^</v-btn
                        ></span
                      >
                    </v-btn>
                  </v-card-actions>
                </v-card>
              </v-list-item-content>
            </UIExtensionSlot>
          </v-list-item>
          <div v-if="needEmail" class="payment-box pt-10">
            <UIExtensionSlot
              name="checkout_email_form"
              :update-email="updateEmail"
              :email-updating="emailUpdating"
              :input-email="inputEmail"
              @update:inputEmail="inputEmail = $event"
              @update:emailFormRef="emailFormRef = $event"
            >
              <v-list-item class="ma-0 pa-0">
                <v-list-item-content class="ma-0 pa-0">
                  <v-form ref="emailForm" @submit.prevent="updateEmail">
                    <v-card flat>
                      <v-card-text>
                        <p class="d-flex justify-center text-h5">
                          Contact & Refund Email
                        </p>
                        <p class="d-flex justify-center">
                          請提供您的電子郵件，若有問題，我們會通過該方式聯繫您。
                        </p>
                        <v-text-field
                          v-model="inputEmail"
                          label="您的電子郵件"
                          :rules="[rules.required, rules.email]"
                        />
                      </v-card-text>
                      <v-card-actions class="d-flex justify-center">
                        <v-btn
                          color="primary"
                          type="submit"
                          :loading="emailUpdating"
                          >Continue</v-btn
                        >
                      </v-card-actions>
                    </v-card>
                  </v-form>
                </v-list-item-content>
              </v-list-item>
            </UIExtensionSlot>
          </div>
          <div v-else-if="needAddress" class="payment-box pt-2 address-form">
            <UIExtensionSlot
              name="checkout_address_form"
              :update-additional="updateAdditional"
              :additional-updating="additionalUpdating"
              :input-notes="inputNotes"
              :input-address="inputAddress"
              @update:inputAddress="inputAddress = $event"
              @update:inputNotes="inputNotes = $event"
              @update:additionalFormRef="additionalFormRef = $event"
            >
              <v-list-item class="ma-0 pa-0">
                <v-list-item-content class="ma-0 pa-0">
                  <v-form
                    ref="additionalForm"
                    @submit.prevent="updateAdditional"
                  >
                    <v-card flat>
                      <v-card-text>
                        <p class="d-flex justify-center text-h5">
                          寄送地址與註記
                        </p>
                        <p class="d-flex justify-center">
                          請在下面提供您的送貨地址，您可以留下有關您的訂單的附加註記。
                        </p>
                        <v-textarea
                          v-model="inputAddress"
                          label="送貨地址"
                          :rules="[rules.required]"
                          :rows="3"
                          no-resize
                        />
                        <v-textarea
                          v-model="inputNotes"
                          label="註記"
                          :rows="3"
                          no-resize
                        />
                      </v-card-text>
                      <v-card-actions class="d-flex justify-center">
                        <v-btn
                          color="primary"
                          type="submit"
                          :loading="additionalUpdating"
                          >Continue</v-btn
                        >
                      </v-card-actions>
                    </v-card>
                  </v-form>
                </v-list-item-content>
              </v-list-item>
            </UIExtensionSlot>
          </div>
        </v-list>
      </div>
    </UIExtensionSlot>
  </v-container>
</template>

<script>
import CloseButton from "@/components/CloseButton"
import DisplayField from "@/components/DisplayField"
import MetamaskButton from "@/components/MetamaskButton"
import WalletConnectButton from "@/components/WalletConnectButton"
import UIExtensionSlot from "@/components/UIExtensionSlot"
export default {
  components: {
    CloseButton,
    DisplayField,
    MetamaskButton,
    WalletConnectButton,
    UIExtensionSlot,
  },
  props: {
    showProp: {
      type: Boolean,
      default: true,
    },
    checkoutPage: {
      type: Boolean,
      default: false,
    },
    invoice: {
      type: Object,
      default() {
        return {}
      },
    },
    store: {
      type: Object,
      default() {
        return {}
      },
    },
  },
  data() {
    return {
      showDetails: false,
      selectedCurrency: 0,
      selectedAction: null,
      selectedToCopy: null,
      showMenu: false,
      endDate: new Date(),
      expirationPercentage: 0,
      timerText: "",
      inputEmail: "",
      inputAddress: "",
      inputNotes: "",
      inputPaymentAddress: "",
      addressUpdating: false,
      paymentAddressErrors: [],
      rules: this.$utils.rules,
      emailUpdating: false,
      additionalUpdating: false,
      abiCache: {},
      paymentSelectionFormRef: null,
      tabsRef: null,
      emailFormRef: null,
      additionalFormRef: null,
    }
  },
  computed: {
    zeroAmountInvoice() {
      return parseFloat(this.invoice.price) === 0
    },
    itemv() {
      return this.invoice.payments[this.selectedCurrency]
    },
    currentPrice() {
      return this.$utils.decimalStr(
        this.itemv.amount * this.itemv.rate,
        this.$utils.getDivisibility(this.invoice.price)
      )
    },
    expectedAmount() {
      return this.$utils.decimalStr(
        this.invoice.price / this.itemv.rate,
        this.itemv.divisibility
      )
    },
    networkFeeIncluded() {
      return parseFloat(this.itemv.amount) > parseFloat(this.expectedAmount)
    },
    currentCurrency() {
      return this.itemv.currency
    },
    qrValue() {
      return this.itemv.lightning && this.selectedToCopy === 1
        ? this.itemv.node_id
        : this.itemv.payment_url
    },
    noTabs() {
      return this.invoice.payments.length === 0
    },
    expiringSoon() {
      return this.expirationPercentage >= 75
    },
    mainProgressColor() {
      return this.expiringSoon ? "#d32f2f" : "#388e3c"
    },
    backgroundProgressColor() {
      return this.expiringSoon ? "red" : "green"
    },
    currencySelectClass() {
      return this.invoice.payments.length > 1
        ? { multipleCurrency: true, rounded: true }
        : {}
    },
    rateClass() {
      const base = { "font-weight-regular": true, "align-right": true }
      return this.zeroAmountInvoice
        ? { ...base, "text-subtitle-2": true }
        : { ...base, "text-caption": true }
    },
    paymentURL() {
      return this.itemv.lightning
        ? `lightning:${this.itemv.payment_url}`
        : this.itemv.payment_url
    },
    logoURL() {
      let url = `${this.STATIC_PATH}/checkout-logo.png`
      if (
        this.store.checkout_settings &&
        this.store.checkout_settings.custom_logo_link
      )
        url = this.store.checkout_settings.custom_logo_link
      return url
    },
    showRecommendedFee() {
      return (
        this.store.checkout_settings &&
        this.store.checkout_settings.show_recommended_fee &&
        this.itemv.recommended_fee !== 0
      )
    },
    isEthPaymentMethod() {
      return this.itemv.payment_url.startsWith("ethereum:")
    },
    needEmail() {
      return (
        this.store.checkout_settings &&
        this.store.checkout_settings.email_required &&
        !this.invoice.buyer_email
      )
    },
    needAddress() {
      return (
        this.store.checkout_settings &&
        this.store.checkout_settings.ask_address &&
        !this.invoice.shipping_address
      )
    },
  },
  watch: {
    showProp(val) {
      if (!val) {
        this.selectedAction = null
        this.selectedToCopy = null
        this.selectedCurrency = 0
      }
    },
    selectedCurrency(val) {
      this.selectedAction = null
      this.selectedToCopy = null
      this.inputPaymentAddress = ""
      this.paymentAddressErrors = []
      this.fetchTokenABI()
    },
  },
  beforeMount() {
    this.$bus.$on("showDetails", () => {
      this.showDetails = true
    })
  },
  beforeDestroy() {
    this.$bus.$off("showDetails")
  },
  mounted() {
    this.fetchTokenABI()
    const date = new Date()
    date.setSeconds(date.getSeconds() + this.invoice.time_left)
    this.endDate = date
    this.startProgressTimer()
  },
  methods: {
    fetchTokenABI() {
      if (!this.itemv) return
      if (!(this.currentCurrency in this.abiCache)) {
        this.$axios
          .get(`/cryptos/tokens/${this.currentCurrency}/abi`)
          .then((res) => {
            this.abiCache[this.currentCurrency] = res.data
          })
      }
    },
    startProgressTimer() {
      const timeLeftS = this.endDate
        ? (this.endDate.getTime() - new Date().getTime()) / 1000
        : this.invoice.time_left
      this.expirationPercentage =
        100 - (timeLeftS / this.invoice.expiration_seconds) * 100
      this.timerText = this.updateTimerText(timeLeftS)
      if (this.expirationPercentage < 100) {
        setTimeout(this.startProgressTimer, 500)
      }
    },
    updateTimerText(timer) {
      if (timer >= 0) {
        let minutes = parseInt(timer / 60, 10)
        minutes = minutes < 10 ? "0" + minutes : minutes
        let seconds = parseInt(timer % 60, 10)
        seconds = seconds < 10 ? "0" + seconds : seconds
        return minutes + ":" + seconds
      } else {
        return "00:00"
      }
    },
    showPopup() {
      this.$emit("closedialog")
      window.bitcart.showInvoice(this.invoice.id)
    },
    checkout(id) {
      if (!id) {
        id = this.qrItem.id
      }
      this.$router.push({ path: `/i/${id}` })
    },
    updateEmail() {
      const ref = this.emailFormRef || this.$refs.emailForm
      if (!ref.validate()) return
      this.emailUpdating = true
      this.$axios
        .patch(`/invoices/${this.invoice.id}/customer`, {
          buyer_email: this.inputEmail,
        })
        .then((r) => {
          this.emailUpdating = false
          this.$emit("update:invoice", {
            ...this.invoice,
            buyer_email: this.inputEmail,
          })
        })
    },
    updateAdditional() {
      const ref = this.additionalFormRef || this.$refs.additionalForm
      if (!ref.validate()) return
      this.additionalUpdating = true
      this.$axios
        .patch(`/invoices/${this.invoice.id}/customer`, {
          notes: this.inputNotes,
          shipping_address: this.inputAddress,
        })
        .then((r) => {
          this.additionalUpdating = false
          this.$emit("update:invoice", {
            ...this.invoice,
            notes: this.inputNotes,
            shipping_address: this.inputAddress,
          })
        })
    },
    updatePaymentDetails(address) {
      if (typeof address === "undefined") {
        const ref =
          this.paymentSelectionFormRef || this.$refs.paymentSelectionForm
        if (!ref.validate()) return
        address = this.inputPaymentAddress
      }
      this.addressUpdating = true
      this.$axios
        .patch(`/invoices/${this.invoice.id}/details`, {
          id: this.itemv.id,
          address,
        })
        .then((r) => {
          this.addressUpdating = false
          this.paymentAddressErrors = []
          const payments = this.invoice.payments
          payments[this.selectedCurrency].user_address = address
          this.$emit("update:invoice", { ...this.invoice, payments })
          // NOTE: a nasty hack to fix vuetify tabs not detecting movements
          const ref = this.tabsRef || this.$refs.tabs
          setTimeout(() => ref.onResize(), 100)
        })
        .catch((err) => {
          this.addressUpdating = false
          if (err.response && err.response.data.detail === "Invalid address") {
            this.paymentAddressErrors = ["Invalid address"]
          }
        })
    },
    getAmountClass(textLen) {
      if (textLen <= 25) return { "text-h5": true }
      else return { "text-h6": true, "font-weight-regular": true }
    },
    openInWallet(url) {
      window.open(url)
    },
  },
}
</script>

<style scoped>
.address-form,
.v-tabs-items.payment-box .v-window-item {
  height: 450px !important;
  overflow-y: auto;
}
.payment-box,
.v-tabs-items.payment-box .v-window-item {
  height: 450px !important;
  overflow-y: auto;
}
.v-application.theme--light .activeTab {
  background: #f5f5f5;
}
.v-application.theme--dark .activeTab {
  background: #424242;
}
.v-list-item__title.align-right {
  align-self: flex-end;
}
.multipleCurrency {
  border-width: 1px;
  border-style: solid;
  padding: 6px;
  overflow: auto !important;
}
.v-application.theme--light .multipleCurrency {
  border-color: rgba(0, 0, 0, 0.12);
}
.v-application.theme--dark .multipleCurrency {
  border-color: rgba(255, 255, 255, 0.12);
}
</style>
