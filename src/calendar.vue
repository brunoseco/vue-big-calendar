<template>
    <div class="comp-full-calendar">
        <!-- header pick month -->
        <fc-header :current-month="currentMonth"
                   :first-day="firstDay"
                   :locale="locale"
                   @change="emitChangeMonth">

            <div slot="header-left">
                <slot name="fc-header-left">
                </slot>
            </div>

            <div slot="header-right">
                <slot name="fc-header-right">
                </slot>
            </div>
        </fc-header>

        <div class="full-calendar-body">
            <div class="weeks">
                <strong class="week" v-for="dayIndex in 7">{{ (dayIndex - 1) | localeWeekDay(firstDay, locale) }}</strong>
            </div>
            <div class="dates" ref="dates">
                <div class="dates-bg">
                    <div class="week-row" v-for="week in currentDates">
                        <div class="day-cell" v-for="day in week" :class="{'today' : day.isToday, 'not-cur-month' : !day.isCurMonth}">
                            <p class="day-number">{{ day.monthDay }}</p>
                        </div>
                    </div>
                </div>

                <div class="dates-events">
                    <div class="events-week" v-for="week in currentDates">
                        <div class="events-day" v-for="day in week" track-by="$index" :class="{'today' : day.isToday, 'not-cur-month' : !day.isCurMonth}" @click.stop="dayClick(day.date, $event)">
                            <p class="day-number">{{day.monthDay}}</p>
                            <div class="event-box">
                                <event-card v-for="event in day.events" :key="event.title" :event="event" :date="day.date" :firstDay="firstDay" v-show="isLimited == false || event.cellIndex <= eventLimit" @click="eventClick"></event-card>
                                <p v-if="isLimited == true && day.events.length > eventLimit"
                                   class="more-link" @click.stop="selectThisDay(day, $event)">
                                    + {{day.events[day.events.length -1].cellIndex - eventLimit}} mais
                                </p>
                            </div>
                        </div>
                    </div>
                </div>

                <div class="more-events" v-show="showMore"
                     :style="{left: morePos.left + 'px', top: morePos.top + 'px'}">
                    <div class="more-header">
                        <span class="title">{{ moreTitle(selectDay.date) }}</span>
                        <span class="close" @click.stop="showMore = false">x</span>
                    </div>
                    <div class="more-body">
                        <ul class="body-list">
                            <li v-for="event in selectDay.events"
                                v-show="event.isShow" class="body-item">
                                <event-card :event="event" :date="selectDay.date" :firstDay="true" @click="eventClick"></event-card>
                            </li>
                        </ul>
                    </div>
                </div>

                <slot name="body-card">

                </slot>

            </div>
        </div>
    </div>
</template>
<script>
    // import langSets from './dataMap/langSets'
    import dateFunc from './components/dateFunc'
    import moment from 'moment';
    import EventCard from './components/eventCard.vue';



    export default {
        name: "vue-big-calendar",
        props: {
            events: { // events will be displayed on calendar
                type: Array,
                default: []
            },
            locale: {
                type: String,
                default: 'en'
            },
            firstDay: {
                type: Number | String,
                validator(val) {
                    let res = parseInt(val);
                    return res >= 0 && res <= 6
                },
                default: 0
            }
        },
        components: {
            'event-card': EventCard,
            'fc-header': require('./components/header')
        },
        mounted() {
            this.emitChangeMonth(this.currentMonth);
        },
        data() {
            return {
                currentMonth: moment().startOf('month'),
                isLimited: false,
                eventLimit: 4,
                showMore: false,
                morePos: {
                    top: 0,
                    left: 0
                },
                selectDay: {}
            }
        },
        computed: {
            currentDates() {
                return this.getCalendar()
            }
        },
        methods: {
            emitChangeMonth(firstDayOfMonth) {
                this.currentMonth = firstDayOfMonth;

                let start = dateFunc.getMonthViewStartDate(firstDayOfMonth, this.firstDay);
                let end = dateFunc.getMonthViewEndDate(firstDayOfMonth, this.firstDay);

                this.$emit('changeMonth', start, end, firstDayOfMonth)
            },
            moreTitle(date) {
                if (!date) return '';
                return moment(date).format('ll');
            },
            getCalendar() {
                // calculate 2d-array of each month
                let monthViewStartDate = dateFunc.getMonthViewStartDate(this.currentMonth, this.firstDay);
                let calendar = [];

                for (let perWeek = 0; perWeek < 6; perWeek++) {
                    let week = [];

                    for (let perDay = 0; perDay < 7; perDay++) {
                        week.push({
                            monthDay: monthViewStartDate.date(),
                            isToday: monthViewStartDate.isSame(moment(), 'day'),
                            isCurMonth: monthViewStartDate.isSame(this.currentMonth, 'month'),
                            weekDay: perDay,
                            date: moment(monthViewStartDate),
                            events: this.slotEvents(monthViewStartDate)
                        });

                        monthViewStartDate.add(1, 'day');
                    }

                    calendar.push(week);
                }

                return calendar
            },
            slotEvents(date) {

                // find all events start from this date
                let cellIndexArr = [];
                let thisDayEvents = this.events.filter(day => {

                    let st = moment(day.start);
                    let ed = moment(day.end ? day.end : st);

                    let start = moment(date).startOf('day');
                    let endOf = moment(date).endOf('day');

                    return st.isBetween(start, endOf, null, '[]') || ed.isBetween(start, endOf, null, '[]');
                });

                // sort by duration
                thisDayEvents.sort((a, b) => {
                    if (!a.cellIndex) return 1;
                    if (!b.cellIndex) return -1;
                    return a.cellIndex - b.cellIndex
                });

                // mark cellIndex and place holder
                for (let i = 0; i < thisDayEvents.length; i++) {
                    thisDayEvents[i].cellIndex = thisDayEvents[i].cellIndex || (i + 1);
                    thisDayEvents[i].isShow = true;
                    if (thisDayEvents[i].cellIndex == i + 1 || i > 2) continue;
                    thisDayEvents.splice(i, 0, {
                        title: 'holder',
                        cellIndex: i + 1,
                        start: date.format(),
                        end: date.format(),
                        isShow: false
                    })
                }

                return thisDayEvents
            },
            selectThisDay(day, jsEvent) {
                this.selectDay = day;
                this.showMore = true;
                this.morePos = this.computePos(event.target);
                this.morePos.top -= 100;
                let events = day.events.filter(item => {
                    return item.isShow == true
                });
                this.$emit('moreClick', day.date, events, jsEvent)
            },
            computePos(target) {
                let eventRect = target.getBoundingClientRect();
                let pageRect = this.$refs.dates.getBoundingClientRect();
                return {
                    left: eventRect.left - pageRect.left,
                    top: eventRect.top + eventRect.height - pageRect.top
                }
            },
            dayClick(day, jsEvent) {
                this.$emit('dayClick', day, jsEvent)
            },
            eventClick(event, jsEvent) {
                if (!event.isShow) return;

                jsEvent.stopPropagation();
                let pos = this.computePos(jsEvent.target);
                this.$emit('eventClick', event, jsEvent, pos);
            }
        },
        filters: {
            localeWeekDay(weekday, firstDay, locale) {
                firstDay = parseInt(firstDay);
                const localMoment = moment().locale(locale);
                return localMoment.localeData().weekdaysShort()[(weekday + firstDay) % 7];
            }
        }
    }

</script>
<style lang="scss">
    .comp-full-calendar {
        padding: 20px;
        background: #fff;
        max-width: 960px;
        margin: 0 auto;
    }

        .comp-full-calendar ul, .comp-full-calendar p {
            margin: 0;
            padding: 0;
        }

    .full-calendar-body {
        margin-top: 20px;
    }

        .full-calendar-body .weeks {
            display: flex;
            border-top: 1px solid #e0e0e0;
            border-bottom: 1px solid #e0e0e0;
            border-left: 1px solid #e0e0e0;
        }

            .full-calendar-body .weeks .week {
                flex: 1;
                text-align: center;
                border-right: 1px solid #e0e0e0;
            }

        .full-calendar-body .dates {
            position: relative;
        }

            .full-calendar-body .dates .week-row {
                border-left: 1px solid #e0e0e0;
                display: flex;
            }

                .full-calendar-body .dates .week-row .day-cell {
                    flex: 1;
                    min-height: 220px;
                    padding: 4px;
                    border-right: 1px solid #e0e0e0;
                    border-bottom: 1px solid #e0e0e0;
                }

                    .full-calendar-body .dates .week-row .day-cell .day-number {
                        text-align: right;
                        border-radius: 5px;
                        color: #000;
                        font-weight: bold;
                        margin-right: 5px;
                    }

                    .full-calendar-body .dates .week-row .day-cell.today {
                        background-color: #fcf8e3;
                    }

                    .full-calendar-body .dates .week-row .day-cell.not-cur-month .day-number {
                        color: rgba(0, 0, 0, 0.24);
                    }

            .full-calendar-body .dates .dates-events {
                position: absolute;
                top: 0;
                left: 0;
                z-index: 1;
                width: 100%;
            }

                .full-calendar-body .dates .dates-events .events-week {
                    display: flex;
                }

                    .full-calendar-body .dates .dates-events .events-week .events-day {
                        flex: 1;
                        min-height: 220px;
                        max-height: 218px;
                        overflow-x: hidden;
                        overflow-y: auto;
                        text-overflow: ellipsis;
                    }

                        .full-calendar-body .dates .dates-events .events-week .events-day::-webkit-scrollbar {
                            position: absolute;
                            width: 8px;
                            -webkit-appearance: none;
                        }

                        .full-calendar-body .dates .dates-events .events-week .events-day::-webkit-scrollbar-thumb {
                            background-color: rgba(32, 168, 216, 0.24);
                            background-clip: content-box;
                            border-color: transparent;
                            border-style: solid;
                            border-width: 1px 2px;
                        }

                        .full-calendar-body .dates .dates-events .events-week .events-day .day-number {
                            text-align: right;
                            padding: 4px 5px 4px 4px;
                            opacity: 0;
                        }

                        .full-calendar-body .dates .dates-events .events-week .events-day.not-cur-month {
                            opacity: 0.5;
                        }

                        .full-calendar-body .dates .dates-events .events-week .events-day.not-cur-month .day-number {
                            color: rgba(0, 0, 0, 0.24);
                        }

                        .full-calendar-body .dates .dates-events .events-week .events-day .event-box .event-item {
                            cursor: pointer;
                            font-size: 12px;
                            background-color: #3179AF;
                            margin-bottom: 2px;
                            color: #FFF;
                            padding: 0 0 0 4px;
                            line-height: 18px;
                            white-space: nowrap;
                            overflow: hidden;
                            text-overflow: ellipsis;
                            border-radius: 5px;
                        }

                            .full-calendar-body .dates .dates-events .events-week .events-day .event-box .event-item .event-item-title {
                                font-weight: bold;
                                text-transform: uppercase;
                            }

                            .full-calendar-body .dates .dates-events .events-week .events-day .event-box .event-item .event-item-description {
                            }

                            .full-calendar-body .dates .dates-events .events-week .events-day .event-box .event-item.is-start {
                                margin-left: 4px;
                            }

                            .full-calendar-body .dates .dates-events .events-week .events-day .event-box .event-item.is-end {
                                margin-right: 4px;
                            }

                            .full-calendar-body .dates .dates-events .events-week .events-day .event-box .event-item.is-opacity {
                                opacity: 0;
                            }

                        .full-calendar-body .dates .dates-events .events-week .events-day .event-box .more-link {
                            cursor: pointer;
                            padding-left: 8px;
                            padding-right: 2px;
                            color: rgba(0, 0, 0, 0.38);
                            font-size: 14px;
                        }

            .full-calendar-body .dates .more-events {
                position: absolute;
                width: 300px;
                z-index: 2;
                border: 1px solid #eee;
                box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
            }

                .full-calendar-body .dates .more-events .more-header {
                    background-color: #eee;
                    padding: 5px;
                    display: flex;
                    align-items: center;
                    font-size: 14px;
                }

                    .full-calendar-body .dates .more-events .more-header .title {
                        flex: 1;
                    }

                    .full-calendar-body .dates .more-events .more-header .close {
                        margin-right: 2px;
                        cursor: pointer;
                        font-size: 16px;
                    }

                .full-calendar-body .dates .more-events .more-body {
                    height: 300px;
                    overflow: hidden;
                }

                    .full-calendar-body .dates .more-events .more-body .body-list {
                        height: 298px;
                        padding: 5px;
                        overflow: auto;
                        background-color: #fff;
                    }

                        .full-calendar-body .dates .more-events .more-body .body-list .body-item {
                            font-size: 12px;
                            margin-bottom: 2px;
                            color: #FFF;
                            line-height: 18px;
                            white-space: nowrap;
                            overflow: hidden;
                            text-overflow: ellipsis;
                            border-radius: 5px;
                        }

                            .full-calendar-body .dates .more-events .more-body .body-list .body-item .event-item {
                                padding-left: 4px;
                                padding-right: 4px;
                            }

                                .full-calendar-body .dates .more-events .more-body .body-list .body-item .event-item .event-item-title {
                                    font-weight: bold;
                                }
</style>