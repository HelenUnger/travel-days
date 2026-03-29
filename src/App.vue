<template>
    <header>
        <h1>Travel Days</h1>
    </header>

    <main>
        <div>
            <label for="travel-days-paste">
                paste here:
            </label>
            <textarea id="travel-days-paste"
                v-model="travelDaysInput"
                name="travel_days_paste"
                rows="4"
                cols="50"
            >
            </textarea>
            <button @click.prevent="setTravelDays()">Submit</button>

            <div>
                <h2>Days Display</h2>
                <br />
                <input
                    id="start"
                    v-model="startDate"
                    type="date"
                    name="trip-start"
                    :min="today"
                    max="2029-12-31" />
                <p>Total days allowed: {{ maxDaysAllowed }}</p>
                <br />
                <table>
                    <thead>
                        <tr>
                            <th>Arrival</th>
                            <th>Departure</th>
                            <th>Net Days</th>
                            <th>Gross Days</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="day in groupedTrips"
                            :class="isWithinAYear(day.departure) ? 'highlight' : ''"
                        >
                            <td>
                                {{ formatDate(day.arrival) }}
                            </td>
                            <td>
                                {{ formatDate(day.departure) }}
                            </td>
                            <td class="text-right">
                                {{ day.totalDays }}
                            </td>
                            <td class="text-right">
                                {{ day.daysWithinLastYear }}
                            </td>
                        </tr>
                        <tr>
                            <td colspan="2">Total Days:</td>
                            <td class="text-right">{{ totalDays }}</td>
                        </tr>
                        <tr>
                            <td colspan="3">Total in last 365 days:</td>
                            <td class="text-right">{{ totalDaysInLastYear }}</td>
                        </tr>
                        <tr>
                            <td colspan="3">Remaining days:</td>
                            <td class="text-right">{{ maxDaysAllowed - totalDaysInLastYear }}</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </main>
</template>

<script>
    import moment from 'moment';
    // import { CalendarView, CalendarViewHeader } from "vue-simple-calendar"
    //
    // import "../node_modules/vue-simple-calendar/dist/style.css"
    export default {
        // components: {
        //     CalendarView,
        //     CalendarViewHeader,
        // },

        data() {
            return {
                travelDaysInput: '',
                travelTrips: [],
                startDate: moment().startOf('day').format('YYYY-MM-DD'),
                today: moment().startOf('day').format('YYYY-MM-DD'),
                maxDaysAllowed: 183,
            }
        },

        computed: {
            oneYearAgo() {
                return this.createDate(this.startDate).subtract(1, 'year');
            },

            sortedTrips() {
                return this.travelTrips.sort((a, b) => this.createDate(a.date).isBefore(this.createDate(b.date)));
            },

            groupedTrips() {
                const trips = [];

                this.sortedTrips.forEach(trip => {
                    let tripObject = {
                        arrival: null,
                        departure: null,
                    };

                    if (trips.length > 0) {
                        const lastIndex = trips.length - 1;
                        const lastTrip = trips[lastIndex];

                        if (lastTrip && (lastTrip.arrival === null || lastTrip.departure === null)) {
                            tripObject = lastTrip;
                        }
                    } else {
                        if (this.isCurrentlyInTravel) {
                            tripObject.departure = this.createDate(this.startDate).format('YYYY-MM-DD');
                        }
                    }

                    const newTripObject = {
                        ...tripObject,
                        [trip.type.toLowerCase()]: trip.date,
                    };


                    if (newTripObject.arrival !== null && newTripObject.departure !== null) {
                        newTripObject.totalDays = this.createDate(newTripObject.departure).diff(newTripObject.arrival, 'days') + 1;


                        if (this.oneYearAgo.isBetween(this.createDate(newTripObject.arrival), this.createDate(newTripObject.departure), '[]')) {
                            console.log('Trip partially in last year/current year', newTripObject);
                            // if a trip is partially in last year/current year
                            newTripObject.daysWithinLastYear = this.createDate(newTripObject.departure).diff(this.oneYearAgo, 'days') + 1;
                        } else if (this.createDate(newTripObject.departure).isAfter(this.oneYearAgo)) {
                            console.log('Trip fully in current year', newTripObject);
                            // if a trip is fully in current year
                            newTripObject.daysWithinLastYear = this.createDate(newTripObject.departure).diff(newTripObject.arrival, 'days') + 1;
                        } else {
                            console.log('Trip fully in last year', newTripObject);
                            // if a trip is fully in last year
                            newTripObject.daysWithinLastYear = 0;
                        }

                        trips.length > 0
                            ? trips.splice(trips.length - 1, 1, newTripObject)
                            : trips.push(newTripObject);
                    } else {
                        trips.push(newTripObject);
                    }
                });

                return trips;
            },

            isCurrentlyInTravel() {
                return this.sortedTrips[0].type === 'Arrival';
            },

            totalDays() {
                return this.groupedTrips.reduce((sum, trip) => sum + trip.totalDays, 0);
            },

            totalDaysInLastYear() {
                return this.groupedTrips
                    .filter(trip => this.isWithinAYear(trip.departure))
                    .reduce((total, trip) => total + trip.daysWithinLastYear, 0);
            }
        },

        methods: {
            createDate(date = null) {
                return moment(date).startOf('day');
            },

            setTravelDays() {
                this.travelTrips = this.textToArrayOfObjects(this.travelDaysInput);
            },

            formatDate(date) {
                return this.createDate(date).format('YYYY-MM-DD');
            },

            isWithinAYear(date) {
                return this.createDate(date).isAfter(this.oneYearAgo);
            },

            textToArrayOfObjects(tsv) {
                // 1. Split the string into lines using the newline character
                const lines = tsv.split('\n');

                // 2. Extract the header (keys) from the first line, splitting by tab
                const headers = lines[0].split('\t').map(header => header.trim().toLowerCase());

                // 3. Process the remaining lines to create the array of objects
                const data = lines.slice(1).map(line => {
                    // Split each line into values by the tab character
                    const values = line.split('\t');
                    // Create an object by zipping headers and values
                    const obj = {};
                    headers.forEach((header, index) => {
                        obj[header] = values[index];
                    });

                    return obj;
                });

                return data;
            }
        },
    }
</script>
<style>
    .highlight {
        background-color: #c9beff;
    }

    .text-right {
        text-align: right;
    }
</style>
