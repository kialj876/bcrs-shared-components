<template>
  <v-card flat id="staff-payment-container" class="mt-4 py-8 pr-6 rounded-0">
    <v-row no-gutters>
      <v-col cols="3" class="pl-5" v-if="displaySideLabel">
        <label class="side-label" :class="{'error-text': invalidSection}">Payment</label>
      </v-col>

      <v-col :cols="displaySideLabel ? 9 : 12" class="pl-2">
        <v-radio-group class="payment-group" v-model="paymentOption" hide-details>
          <!-- Cash or Cheque radio button and form -->
          <v-radio id="fas-radio" label="Cash or Cheque" :value="StaffPaymentOptions.FAS" />
          <v-form class="ml-8" ref="fasForm" v-model="fasFormValid">
            <v-text-field
              filled
              id="routing-slip-number-textfield"
              label="Routing Slip Number"
              :value="staffPaymentData.routingSlipNumber"
              :rules="validate ? routingSlipNumberRules : []"
              :disabled="paymentOption === StaffPaymentOptions.BCOL || paymentOption === StaffPaymentOptions.NO_FEE"
              @focus="paymentOption = StaffPaymentOptions.FAS"
              @input="emitStaffPaymentData({ option: StaffPaymentOptions.FAS, routingSlipNumber: $event })"
            />
          </v-form>

          <!-- BC Online radio button and form -->
          <v-radio id="bcol-radio" label="BC Online" :value="StaffPaymentOptions.BCOL" />
          <v-form class="ml-8" ref="bcolForm" v-model="bcolFormValid">
            <v-text-field
              filled
              id="bcol-account-number-textfield"
              label="BC Online Account Number"
              :value="staffPaymentData.bcolAccountNumber"
              :rules="validate ? bcolAccountNumberRules : []"
              :disabled="paymentOption === StaffPaymentOptions.FAS || paymentOption === StaffPaymentOptions.NO_FEE"
              @focus="paymentOption = StaffPaymentOptions.BCOL"
              @input="emitStaffPaymentData({ option: StaffPaymentOptions.BCOL, bcolAccountNumber: $event })"
            />
            <v-text-field
              filled
              id="dat-number-textfield"
              label="DAT Number"
              :value="staffPaymentData.datNumber"
              :rules="validate ? datNumberRules : []"
              :disabled="paymentOption === StaffPaymentOptions.FAS || paymentOption === StaffPaymentOptions.NO_FEE"
              @focus="paymentOption = StaffPaymentOptions.BCOL"
              @input="emitStaffPaymentData({ option: StaffPaymentOptions.BCOL, datNumber: $event })"
            />
            <FolioNumberInput
              ref="folioNumberInputRef"
              :folioNumber="staffPaymentData.folioNumber"
              :disabled="paymentOption === StaffPaymentOptions.FAS || paymentOption === StaffPaymentOptions.NO_FEE"
              @focus="paymentOption = StaffPaymentOptions.BCOL"
              @emitFolioNumber="paymentOption === StaffPaymentOptions.BCOL &&
                emitStaffPaymentData({ option: StaffPaymentOptions.BCOL, folioNumber: $event })"
              validate="true"
            />
          </v-form>

          <!-- No Fee radio button -->
          <v-radio id="no-fee-radio" label="No Fee" :value="StaffPaymentOptions.NO_FEE" />

          <template v-if="displayPriorityCheckbox">
            <v-divider class="mt-5"></v-divider>

            <!-- Priority checkbox -->
            <v-checkbox
              class="mt-2"
              id="priority-checkbox"
              label="Priority (add $100.00)"
              hide-details
              :input-value="staffPaymentData.isPriority"
              :disabled="paymentOption === StaffPaymentOptions.NO_FEE"
              @change="emitStaffPaymentData({ isPriority: !!$event })"
            />
          </template>
        </v-radio-group>
      </v-col>
    </v-row>
  </v-card>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch, Emit } from 'vue-property-decorator'
import { StaffPaymentOptions } from '@bcrs-shared-components/enums'
import { FolioNumberInput } from '@bcrs-shared-components/folio-number-input'
import { FormIF, StaffPaymentIF } from '@bcrs-shared-components/interfaces'

@Component({
  components: { FolioNumberInput }
})
export default class StaffPayment extends Vue {
  // To fix "property X does not exist on type Y" errors, annotate types for referenced components.
  // ref: https://github.com/vuejs/vetur/issues/1414
  // ref: https://github.com/vuejs/vue-class-component/issues/94
  $refs!: Vue['$refs'] & {
    fasForm: FormIF,
    bcolForm: FormIF,
    folioNumberInputRef: FolioNumberInput
  }

  /** Enum for template. */
  readonly StaffPaymentOptions = StaffPaymentOptions

  /** Whether to display side label. */
  @Prop({ default: true })
  readonly displaySideLabel: boolean

  /** Whether to display priority checkbox. */
  @Prop({ default: true })
  readonly displayPriorityCheckbox: boolean

  /** Whether to validate the fields. */
  @Prop({ default: false })
  private validate: boolean

  /** Whether to show invalid section styling (label only). */
  @Prop({ default: false })
  private invalidSection: boolean

  /** Staff Payment Data prop. */
  @Prop({
    default: () => {
      return {
        option: StaffPaymentOptions.NONE,
        routingSlipNumber: null,
        bcolAccountNumber: null,
        datNumber: null,
        folioNumber: null,
        isPriority: false
      }
    }
  })
  private staffPaymentData: StaffPaymentIF

  /** Radio group model property. */
  private paymentOption = StaffPaymentOptions.NONE

  /** FAS form model property. */
  private fasFormValid = false

  /** BCOL form model property. */
  private bcolFormValid = false

  /** Whether this component has been mounted. */
  private isMounted = false

  /** Validation rules for Routing Slip Number. */
  private readonly routingSlipNumberRules: Array<Function> = [
    v => !!v || 'Enter FAS Routing Slip Number',
    v => /^\d{9}$/.test(v) || 'Routing Slip Number must be 9 digits'
  ]

  /** Validation rules for BCOL Account Number. */
  private readonly bcolAccountNumberRules: Array<Function> = [
    v => !!v || 'Enter BC Online Account Number',
    v => /^\d{6}$/.test(v) || 'BC Online Account Number must be 6 digits'
  ]

  /** Validation rules for DAT Number. */
  private readonly datNumberRules: Array<Function> = [
    v => !!v || 'Enter DAT Number',
    v => /^[A-Z]{1}[0-9]{7,9}$/.test(v) || 'DAT Number must be in standard format (eg, C1234567)'
  ]

  /** Called when this component is mounted. */
  private async mounted (): Promise<void> {
    await this.$nextTick()
    this.isMounted = true
  }

  /** Called when payment option (radio group item) has changed. */
  @Watch('paymentOption')
  private onPaymentOptionChanged (val: number): void {
    switch (val) {
      case StaffPaymentOptions.FAS:
        // reset other form
        this.$refs.bcolForm.resetValidation()
        this.$refs.folioNumberInputRef.resetFolioNumberValidation()
        // enable validation for this form
        this.$refs.fasForm.validate()
        // update data
        this.emitStaffPaymentData({ option: StaffPaymentOptions.FAS })
        break

      case StaffPaymentOptions.BCOL:
        // reset other form
        this.$refs.fasForm.resetValidation()
        // enable validation for this form
        this.$refs.bcolForm.validate()
        // update data
        this.emitStaffPaymentData({ option: StaffPaymentOptions.BCOL })
        break

      case StaffPaymentOptions.NO_FEE:
        // reset other forms
        this.$refs.fasForm.resetValidation()
        this.$refs.bcolForm.resetValidation()
        this.$refs.folioNumberInputRef.resetFolioNumberValidation()
        // update data
        this.emitStaffPaymentData({ option: StaffPaymentOptions.NO_FEE, isPriority: false })
        break
    }
  }

  /** Watches for change to FAS form validity. */
  @Watch('fasFormValid')
  private onFasFormValid (val: boolean) {
    // ignore initial condition
    if (!this.isMounted) return
    this.emitValid()
  }

  /** Watches for change to BCOL form validity. */
  @Watch('bcolFormValid')
  private onBcolFormValid (val: boolean) {
    // ignore initial condition
    if (!this.isMounted) return
    this.emitValid()
  }

  /** Watches for changes to Staff Payment Data prop. */
  @Watch('staffPaymentData', { deep: true, immediate: true })
  private async onStaffPaymentDataChanged (val: StaffPaymentIF): Promise<void> {
    this.paymentOption = val.option
    await this.$nextTick()
    this.emitValid()
  }

  /** Emits an event to update the Staff Payment Data prop. */
  @Emit('update:staffPaymentData')
  private emitStaffPaymentData ({
    option = this.staffPaymentData.option,
    routingSlipNumber = this.staffPaymentData.routingSlipNumber || '',
    bcolAccountNumber = this.staffPaymentData.bcolAccountNumber || '',
    datNumber = this.staffPaymentData.datNumber || '',
    folioNumber = this.staffPaymentData.folioNumber || '',
    isPriority = this.staffPaymentData.isPriority || false
  }): StaffPaymentIF {
    // return only the appropriate fields for each option
    switch (option) {
      case StaffPaymentOptions.FAS:
        return { option, routingSlipNumber, isPriority } as StaffPaymentIF

      case StaffPaymentOptions.BCOL:
        return { option, bcolAccountNumber, datNumber, folioNumber, isPriority } as StaffPaymentIF

      case StaffPaymentOptions.NO_FEE:
        return { option } as StaffPaymentIF
    }
  }

  /** Emits an event indicating whether or not this component is valid. */
  @Emit('valid')
  private emitValid (): boolean {
    return (this.fasFormValid ||
      (this.bcolFormValid && this.$refs.folioNumberInputRef.validateFolioNumber()) ||
      (this.staffPaymentData.option === StaffPaymentOptions.NO_FEE)
    )
  }
}
</script>

<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

#staff-payment-container {
  margin-top: 1rem;
  padding-bottom: 0.5rem;
  padding-top: 1rem;
  line-height: 1.2rem;
  font-size: 1rem;
}

.payment-group {
  margin-top: 0;
  padding-top: 0;
  width: 100%;
}

// radio buttons
::v-deep .v-input--selection-controls__input >.v-icon {
  color: $gray7 !important;
}

// radio labels and checkbox labels
::v-deep .v-input--selection-controls__input + .v-label {
  color: $gray7 !important;
}

// text field labels
::v-deep .v-text-field__slot .theme--light.v-label {
  color: $gray7 !important;
}

// text input
::v-deep .v-text-field__slot input {
  color: $gray9 !important;
}
</style>
