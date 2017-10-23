<template>
    <div class="event-item"
         :class="cssClasses"
         v-tooltip.left="tooltip"
         v-bind:style="{ 'background-color': event.color }"
         @click="$emit('click', event, $event)">
        <span class="event-item-title">{{event.title}}</span>
        <p class="event-item-description">{{event.description}}</p>
    </div>
</template>

<script>

    import Vue from 'vue'
    import VTooltip from 'v-tooltip'
    Vue.use(VTooltip)

    import moment from 'moment'

    export default {
        props: ['event', 'date', 'firstDay'],
        computed: {
            cssClasses() {
                let cssClasses = this.event.cssClass;

                if (!Array.isArray(cssClasses)) {
                    cssClasses = [cssClasses];
                } else {
                    cssClasses = Array.from(cssClasses);
                }

                if (this.start.isSame(this.date, 'day')) {
                    cssClasses.push('is-start');
                }

                if (this.end.isSame(this.date, 'day')) {
                    cssClasses.push('is-end');
                }

                if (!this.event.isShow) {
                    cssClasses.push('is-opacity');
                }

                return cssClasses.join(' ');
            },
            showTitle() {
                return (this.date.day() == this.firstDay || this.start.isSame(this.date, 'day'));
            },
            start() {
                return moment(this.event.start);
            },
            end() {
                return moment(this.event.end);
            },
            tooltip() {
                if (this.start.format("HH:mm") == this.end.format("HH:mm"))
                    return moment(this.event.start).format("HH:mm");

                return moment(this.start).format("HH:mm") + ' - ' + moment(this.end).format("HH:mm");
            },
        },
    }
</script>