<template>
  <div>
    <div class="datepicker__tooltip" 
      v-if="showTooltip && this.options.hoveringTooltip" 
      v-html="tooltipMessageDisplay">
    </div>
    <div class="datepicker__month-day" 
      @click.prevent.stop="dayClicked(date)" 
      @keyup.enter.prevent.stop="dayClicked(date)" 
      v-text="dayNumber" 
      :class="dayClass" 
      :style="isToday ? currentDateStyle : ''" 
      :tabindex="tabIndex" 
      ref="day">
    </div>
</div>
</template>

<script>
import fecha from 'fecha';

import Helpers from './helpers.js'

export default {
  name: 'Day',

  props: {
    isOpen: {
      type: Boolean,
      required: true,
    },
    sortedDisabledDates: {
      type: Array
    },
    options: {
      type: Object
    },
    checkIn: {
      type: Date
    },
    checkOut: {
      type: Date
    },
    hoveringDate: {
      type: Date
    },
    mounseOverFunction: {
      type: Function
    },
    belongsToThisMonth: {
      type: Boolean
    },
    activeMonthIndex: {
      type: Number
    },
    date: {
      type: Date
    },
    dayNumber: {
      type: String
    },
    nextDisabledDate: {
      type: [Date, Number, String]
    },
    hoveringTooltip: {
      default: true,
      type: Boolean
    },
    tooltipMessage: {
      default: null,
      type: String
    },
    currentDateStyle:{
      required: true,
    }
  },

  data() {
    return {
      isHighlighted: false,
      isDisabled: false,
      allowedCheckoutDays: [],
      currentDate: new Date(),
    };
  },

  computed: {
    tabIndex() {
      if (!this.isOpen || !this.belongsToThisMonth || this.isDisabled || !this.isClickable()) {
        return -1;
      }
      return 0
    },
    nightsCount() {
      return this.countDays(this.checkIn, this.hoveringDate);
    },
    tooltipMessageDisplay() {
      return this.tooltipMessage
      ? this.tooltipMessage
      : `${this.nightsCount} ${this.nightsCount !== 1 ?
              this.options.i18n['nights'] : this.options.i18n['night']}`
    },
    showTooltip() {
      return  !this.isDisabled &&
              this.date == this.hoveringDate &&
              this.checkIn !== null &&
              this.checkOut == null;
    },

    isToday() {
      return this.compareDay(this.currentDate, this.date) == 0;
    },

    dayClass(){
      if (this.belongsToThisMonth) {
        // If the calendar has a minimum number of nights
        if ( !this.isDisabled &&
             this.compareDay(this.date, this.checkIn) == 1 &&
             this.options.minNights > 0 &&
             this.compareDay(
                this.date,
                this.addDays(this.checkIn, this.options.minNights)
              ) == -1) {
            return 'datepicker__month-day--selected datepicker__month-day--out-of-range'
        }

        // If the calendar has allowed ranges
        if (this.options.allowedRanges.length !== 0) {
          if ( !this.isDisabled && this.checkIn !== null && this.checkOut == null ) {
            // If the day is one of the allowed check out days and is not highlighted
            if ( this.allowedCheckoutDays.some((i) => this.compareDay(i, this.date) == 0 && !this.isHighlighted) ) {
              return 'datepicker__month-day--allowed-checkout'
            }
            // If the day is one of the allowed check out days and is highlighted
            if ( this.allowedCheckoutDays.some((i) => this.compareDay(i, this.date) == 0 && this.isHighlighted) ) {
              return 'datepicker__month-day--selected datepicker__month-day--allowed-checkout'
            }
            // If the day is not one of the allowed Checkout Days and is highlighted
            if ( !(this.allowedCheckoutDays.some((i) => this.compareDay(i, this.date) == 0 )) && this.isHighlighted) {
              return 'datepicker__month-day--out-of-range datepicker__month-day--selected'
            }
            else {
              return 'datepicker__month-day--out-of-range'
            }
          }
        }
        // Highlight the selected dates and prevent the user from selecting
        // the same date for checkout and checkin
        if ( this.checkIn !== null &&
            ( fecha.format(this.checkIn, 'YYYYMMDD') == fecha.format(this.date, 'YYYYMMDD') )
          ) {
          if (this.options.minNights == 0) {
            return "datepicker__month-day--first-day-selected c-bg-fe5101"
          } else {
            return "datepicker__month-day--first-day-selected c-bg-fe5101"
          }
        }
        if ( this.checkOut !== null ) {
          if ( fecha.format(this.checkOut, 'YYYYMMDD') == fecha.format(this.date, 'YYYYMMDD') ) {
            return "datepicker__month-day--last-day-selected c-bg-fe5101"
          }
        }
        // Only highlight dates that are not disabled
        if ( this.isHighlighted && !this.isDisabled) { return " datepicker__month-day--selected"}
        if ( this.isDisabled ) { return "datepicker__month-day--disabled" }
      }

      else if ( !this.belongsToThisMonth ) { return "datepicker__month-day--hidden" }
      else {  return "datepicker__month-day--valid" }
    },
  },

  watch: {
    hoveringDate(date) {
      if ( this.checkIn !== null  && this.checkOut == null && this.isDisabled == false) {
        this.isDateLessOrEquals(this.checkIn, this.date) &&
        this.isDateLessOrEquals(this.date, this.hoveringDate) ?
        this.isHighlighted = true : this.isHighlighted = false
      }
      if( this.checkIn !== null  && this.checkOut == null && this.allowedCheckoutDays.length !== 0){

      }
    },
    activeMonthIndex(index) {
      this.checkIfDisabled()
      this.checkIfHighlighted()
      if ( this.checkIn !== null  && this.checkOut !== null ) {
          this.isDateLessOrEquals(this.checkIn, this.date) &&
          this.isDateLessOrEquals(this.date, this.checkOut) ?
          this.isHighlighted = true : this.isHighlighted = false
      } else if ( this.checkIn !== null  && this.checkOut == null ) {
        this.disableNextDays()
      } else {
        return
      }

    },
    nextDisabledDate() {
      this.disableNextDays();
    },
    checkIn(date) {
      this.createAllowedCheckoutDays(date);
      this.checkIfDisabled();
      this.checkIfHighlighted();
    },
  },

  methods: {
    ...Helpers,

    isClickable() {
      if (this.$refs && this.$refs.day) {
        return getComputedStyle(this.$refs.day).pointerEvents !== 'none';
      } else {
        return true;
      }
    },

    compareDay(day1, day2) {
      const date1 = fecha.format(new Date(day1), 'YYYYMMDD');
      const date2 = fecha.format(new Date(day2), 'YYYYMMDD');

      if (date1 > date2) { return 1; }

      else if (date1 == date2) { return 0; }

      else if (date1 < date2) { return -1; }
    },

    dayClicked(date) {
      if (this.isDisabled || !this.isClickable()) {
        return
      }
      else {
        if (this.options.allowedRanges.length !== 0) {
          this.createAllowedCheckoutDays(date);
        }

        let nextDisabledDate =
          (this.options.maxNights ? this.addDays(this.date, this.options.maxNights) : null) ||
          this.allowedCheckoutDays[this.allowedCheckoutDays.length-1] ||
          this.getNextDate(this.sortedDisabledDates, this.date) ||
          this.nextDateByDayOfWeekArray(this.options.disabledDaysOfWeek, this.date) ||
          Infinity;

        if (this.options.enableCheckout) { nextDisabledDate = Infinity; }

        this.$emit('day-clicked', { date, nextDisabledDate });
      }
    },

    compareEndDay() {
      if (this.options.endDate !== Infinity) {
        return ( this.compareDay(this.date, this.options.endDate) == 1 )
      }
    },

    checkIfDisabledInRotation() {

      //console.log('ROTATION');
      //console.log(fecha.format(this.date, 'dddd'));

      if(this.options.rotations === undefined){
        return false;
      }
      if(this.options.rotations === null){
        return false;
      }

      //end date
      if(this.checkIn !== null && this.checkOut === null){

        if(this.options.rotations.rotationReturn === undefined){
          return false;
        }

        if(this.options.rotations.rotationReturn.length === 0){
          return true;
        }

        //not in rotation period
        if(!this.options.rotations.rotationReturn.some((i) => 
          (this.compareDay(this.date, i.rotationStartingDate) == 1 ||  this.compareDay(this.date, i.rotationStartingDate) == 0) &&
          (this.compareDay(this.date, i.rotationEndingDate) == -1 || this.compareDay(this.date, i.rotationEndingDate) == 0))){
            return true;
        }

        var rotationReturn = this.options.rotations.rotationReturn.filter((i) => 
          (this.compareDay(this.date, i.rotationStartingDate) == 1 || this.compareDay(this.date, i.rotationStartingDate) == 0) &&
          (this.compareDay(this.date, i.rotationEndingDate) == -1 || this.compareDay(this.date, i.rotationEndingDate) == 0))[0];

        //in rotation period but not an allowed day
        if(!rotationReturn.rotationDays.some((i) => i == fecha.format(this.date, 'dddd'))){
            return true;
        }
      }
      else{ //start date

        if(this.options.rotations.rotationDeparture === undefined){
          return false;
        }

        if(this.options.rotations.rotationDeparture.length === 0){
          return true;
        }

        //not in rotation period
        if(!this.options.rotations.rotationDeparture.some((i) => 
          (this.compareDay(this.date, i.rotationStartingDate) == 1 || this.compareDay(this.date, i.rotationStartingDate) == 0) && 
          (this.compareDay(this.date, i.rotationEndingDate) == -1) || this.compareDay(this.date, i.rotationEndingDate) == 0)){
            return true;
        }

        var rotationDeparture = this.options.rotations.rotationDeparture.filter((i) => 
          (this.compareDay(this.date, i.rotationStartingDate) == 1 || this.compareDay(this.date, i.rotationStartingDate) == 0) && 
          (this.compareDay(this.date, i.rotationEndingDate) == -1 || this.compareDay(this.date, i.rotationEndingDate) == 0))[0];

        //in rotation period but not an allowed day
        if(!rotationDeparture.rotationDays.some((i) => i == fecha.format(this.date, 'dddd'))){
            return true;
        }
      }
    },

    checkIfDisabled() {
      this.isDisabled =
        // If this day is equal any of the disabled dates
        (this.sortedDisabledDates ? this.sortedDisabledDates.some((i) =>
          this.compareDay(i, this.date) == 0
        ) : null)
        // Or is before the start date
        || this.compareDay(this.date, this.options.startDate) == -1
        // Or is before checkin when selecting checkout
        || this.checkIn !== null && this.checkOut === null && this.compareDay(this.date, this.checkIn) == -1
        // Or is after the end date
        || this.compareEndDay()
        // Or is in one of the disabled days of the week
        || this.options.disabledDaysOfWeek.some((i) =>
          i == fecha.format(this.date, 'dddd'))
        // Or is on of the disabled days in the rotation
        || this.checkIfDisabledInRotation();
        // Handle checkout enabled
        if ( this.options.enableCheckout ) {
          if ( this.compareDay(this.date, this.checkIn) == 1 &&
               this.compareDay(this.date, this.checkOut) == -1 ) {
                this.isDisabled = false;
          }
        }
    },

    checkIfHighlighted(){
      if ( this.checkIn !== null  && this.checkOut !== null && this.isDisabled == false) {
        this.isDateLessOrEquals(this.checkIn, this.date) &&
        this.isDateLessOrEquals(this.date, this.checkOut) ?
        this.isHighlighted = true : this.isHighlighted = false
      }
    },

    createAllowedCheckoutDays(date){
      this.allowedCheckoutDays = [];
      this.options.allowedRanges.forEach((i) =>
        this.allowedCheckoutDays.push(this.addDays(date, i))
      )
      this.allowedCheckoutDays.sort((a, b) =>  a - b );
    },

    disableNextDays(){
      if ( !this.isDateLessOrEquals(this.date, this.nextDisabledDate)
            && this.nextDisabledDate !== Infinity) {
              this.isDisabled = true;
      }
      else if ( this.isDateLessOrEquals(this.date, this.checkIn) ) {
        this.isDisabled = true;
      }
      if ( this.compareDay(this.date, this.checkIn) == 0 && this.options.minNights == 0) {
        this.isDisabled = false;
      }
      if (this.isDateLessOrEquals(this.checkIn, this.date) && this.options.enableCheckout ){
        this.isDisabled = false
      }
      else {
        return
      }
    },
  },

  beforeMount(){
    this.checkIfDisabled()
    this.checkIfHighlighted()
  },
}
</script>
