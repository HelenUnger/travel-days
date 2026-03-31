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
                <p>Total days allowed: {{ maxDaysAllowed }}</p>
                <br />
                <table>
                    <thead>
                        <tr>
                            <th>Arrival</th>
                            <th>Departure</th>
                            <th>Net Days</th>
                            <th>Gross Days</th>
                            <th></th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-if="allTrips.length === 0">
                            <td colspan="4">No trips found</td>
                        </tr>
                        <tr>
                            <td colspan="4">
                                <a
                                    class="btn-link text-center"
                                    @click="addTrip()"
                                >
                                    + Add Trip
                                </a>
                            </td>
                        </tr>
                        <tr v-for="(day, index) in allTrips"
                            :class="isWithinAYear(day.departure) ? 'highlight' : ''"
                        >
                            <td>
                                <template v-if="day.custom">
                                    <input
                                        :id="`custom-start-day-${index}`"
                                        v-model="day.arrival"
                                        type="date"
                                        :min="getMinDate(index)"
                                        :max="day.departure"
                                        @change="recalculateCustom()"
                                    />
                                </template>
                                <template v-else>
                                    {{ formatDate(day.arrival) }}
                                </template>
                            </td>
                            <td>
                                <template v-if="day.custom">
                                    <input
                                        :id="`custom-end-day-${index}`"
                                        v-model="day.departure"
                                        type="date"
                                        :min="day.arrival"
                                        :max="getMaxDate(index)"
                                        @change="recalculateCustom()"
                                    />
                                </template>
                                <template v-else-if="day.partialTravel">
                                    <input
                                        id="start"
                                        v-model="startDate"
                                        type="date"
                                        name="trip-start"
                                        :min="today"
                                        max="2029-12-31"
                                        @change="updateAllTrips()"
                                    />
                                </template>
                                <template v-else>
                                    {{ formatDate(day.departure) }}
                                </template>
                            </td>
                            <td class="text-right">
                                {{ day.totalDays }}
                            </td>
                            <td class="text-right">
                                {{ day.daysWithinLastYear }}
                            </td>
                            <td>
                                <button
                                    v-if="day.custom"
                                    :id="`delete-day-${index}`"
                                    @click="removeTrip(index)"
                                    class="btn btn-danger"
                                >
                                    Delete
                                </button>
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
        data() {
            return {
                travelDaysInput: '',
                travelTrips: [],
                startDate: moment().startOf('day').format('YYYY-MM-DD'),
                today: moment().startOf('day').format('YYYY-MM-DD'),
                maxDaysAllowed: 183,
                customTrips: [],
                allTrips: []
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

                this.sortedTrips.forEach((trip, index) => {
                    let tripObject = {
                        id: index,
                        arrival: null,
                        departure: null,
                        custom: 0,
                        partialTravel: 0,
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
                            tripObject.partialTravel = 1;
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
                return this.allTrips.reduce((sum, trip) => sum + trip.totalDays, 0);
            },

            totalDaysInLastYear() {
                return this.allTrips
                    .filter(trip => this.isWithinAYear(trip.departure))
                    .reduce((total, trip) => total + trip.daysWithinLastYear, 0);
            }
        },

        created() {
            // Retrieve data
            // Parse the JSON string back into an object
            this.travelDaysInput = JSON.parse(localStorage.getItem('trip-input'));
        },

        methods: {
            addTrip() {
                this.customTrips.splice(0, 0, {
                    id: `custom-${this.customTrips.length}`,
                    departure: this.createDate(),
                    arrival: this.createDate().add(1, 'days'),
                    daysWithinLastYear: 1,
                    totalDays: 1,
                    custom: 1,
                    partialTravel: 0
                });

                this.updateAllTrips();
            },

            removeTrip(index) {
                this.customTrips.splice(index, 1);
                this.updateAllTrips();
            },

            updateAllTrips() {
                this.allTrips = this.customTrips.concat(this.groupedTrips);
            },

            recalculateCustom() {
                this.customTrips = this.customTrips.map(trip => {
                    if (trip.arrival == null || trip.departure == null ) {
                        return trip;
                    }

                    trip.totalDays = this.createDate(trip.departure).diff(this.createDate(trip.arrival), 'days') + 1;

                    if (this.oneYearAgo.isBetween(this.createDate(trip.arrival), this.createDate(trip.departure), '[]')) {
                        // if a trip is partially in last year/current year
                        trip.daysWithinLastYear = this.createDate(trip.departure).diff(this.oneYearAgo, 'days') + 1;
                    } else if (this.createDate(trip.departure).isAfter(this.oneYearAgo)) {
                        // if a trip is fully in current year
                        trip.daysWithinLastYear = this.createDate(trip.departure).diff(trip.arrival, 'days') + 1;
                    } else {
                        // if a trip is fully in last year
                        trip.daysWithinLastYear = 0;
                    }

                    return trip;
                });

                this.allTrips = this.customTrips.concat(this.groupedTrips);
            },

            createDate(date = null) {
                return moment(date).startOf('day');
            },

            setTravelDays() {
                this.travelTrips = this.textToArrayOfObjects(this.travelDaysInput);
                localStorage.setItem('trip-input', JSON.stringify(this.travelDaysInput));

                this.allTrips = this.groupedTrips;
            },

            getMinDate(index) {
                if (this.allTrips[index + 1] === undefined) {
                    return null;
                }

                return moment(this.allTrips[index + 1].departure).add(1, 'days').format('YYYY-MM-DD');
            },

            getMaxDate(index) {
                if (this.allTrips[index - 1] === undefined) {
                    return null;
                }

                return moment(this.allTrips[index - 1].arrival).subtract(1, 'days').format('YYYY-MM-DD');
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

    .btn-primary {
        background-color: #007bff;
        border-color: #007bff;
    }

    .btn-danger {
        background-color: #dc3545;
        border-color: #dc3545;
    }

    .btn {
        display: inline-block;
        font-weight: 400;
        text-align: center;
        white-space: nowrap;
        vertical-align: middle;
        border-radius: .2rem;
    }

    .btn-link {
        font-weight: 400;
        color: #007bff;
        text-decoration: none;
        cursor: pointer;
    }
</style>
