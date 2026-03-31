<template>
    <header>
        <h1 class="text-center">Travel Days</h1>
    </header>

    <main>
        <div class="m-auto text-center">
            <div v-if="showPasteInput">
                <p>To start, either paste your travel days or enter custom trips.</p>
                <p class="mt-3">
                    To get your past travel days, visit U.S. Customs and Border Protection I94
                    <br />
                    travel history site here:
                    <a href="https://i94.cbp.dhs.gov/search/history-search"
                       class="btn-link"
                    >
                        https://i94.cbp.dhs.gov/search/history-search
                    </a>
                </p>

                <p>
                    After entering your passport information and retrieving the data,
                    <br />
                    use your cursor to highlight the table headings and all the rows.
                    <br />
                    Then copy and paste into the below input.
                </p>

                <p>*DO NOT REFORMAT AFTER PASTING*</p>


                <textarea id="travel-days-paste"
                    v-model="travelDaysInput"
                    name="travel_days_paste"
                    rows="4"
                    cols="50"
                    class="mb-3 mt-3 m-0-auto d-block"
                >
                </textarea>
                <button @click.prevent="setTravelDays()">
                    Submit
                </button>

                <p>------------- OR ------------</p>

                <a
                    class="btn-link text-center"
                    @click="enterCustom()"
                >
                    Enter custom trips
                </a>

            </div>
            <div v-else class="m-0-auto">
                <button @click.prevent="showPasteInput = true">Back</button>
                <p>Total days allowed: {{ maxDaysAllowed }}</p>
                <br />
                <table id="trips-table" class="m-0-auto text-left">
                    <thead>
                        <tr>
                            <th>Arrival</th>
                            <th>Departure</th>
                            <th>Total Days</th>
                            <th>Within Past 12 Months</th>
                            <th></th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-if="allTrips.length === 0">
                            <td colspan="4" class="text-center">No trips found</td>
                        </tr>
                        <tr>
                            <td colspan="4" class="text-center">
                                <a
                                    class="btn-link"
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
    export default {
        data() {
            return {
                showPasteInput: true,
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
                            // if a trip is partially in last year/current year
                            newTripObject.daysWithinLastYear = this.createDate(newTripObject.departure).diff(this.oneYearAgo, 'days') + 1;
                        } else if (this.createDate(newTripObject.departure).isAfter(this.oneYearAgo)) {
                            // if a trip is fully in current year
                            newTripObject.daysWithinLastYear = this.createDate(newTripObject.departure).diff(newTripObject.arrival, 'days') + 1;
                        } else {
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
                let arrival = moment().startOf('day').add(1, 'days').format('YYYY-MM-DD');
                let departure = moment().startOf('day').add(2, 'days').format('YYYY-MM-DD');

                if (this.customTrips.length > 0) {
                    arrival = moment(this.customTrips[0].departure).add(1, 'days').format('YYYY-MM-DD');
                    departure = moment(arrival).add(1, 'days').format('YYYY-MM-DD');
                }

                this.customTrips.splice(0, 0, {
                    id: `custom-${this.customTrips.length}`,
                    arrival,
                    departure,
                    daysWithinLastYear: 2,
                    totalDays: 2,
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

                this.showPasteInput = false;
                this.allTrips = this.groupedTrips;
            },

            enterCustom() {
                this.showPasteInput = false;
                this.allTrips = [];
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
    #trips-table {
        min-width: 700px;
        border-collapse: collapse;
    }
</style>
