<template>
  <v-row class="fill-height">
    <v-col>
      <v-sheet height="64">
        <v-toolbar flat color="white">
          <v-btn color="primary" class="mr-4" @click="handleAddDialog()">Add new...</v-btn>
          <v-btn outlined class="mr-4" @click="setToday">Today</v-btn>
          <v-btn fab text small @click="prev">
            <v-icon small>mdi-chevron-left</v-icon>
          </v-btn>
          <v-toolbar-title>{{ title }}</v-toolbar-title>
          <v-btn fab text small @click="next">
            <v-icon small>mdi-chevron-right</v-icon>
          </v-btn>
          <v-spacer></v-spacer>
          <v-form @submit.prevent="setApiPassw()">
            <v-text-field v-model="apiPassword" type="password" label="API Password" right bottom></v-text-field>
            <v-btn type="submit" outline right bottom>Set API</v-btn>
          </v-form>
          <v-btn @click="getEvents()" outlined right bottom>Reload</v-btn>
          <v-menu bottom right>
            <template v-slot:activator="{ on }">
              <v-btn outlined v-on="on">
                <span>{{ typeToLabel[calendarType] }}</span>
                <v-icon right>mdi-menu-down</v-icon>
              </v-btn>
            </template>
            <v-list>
              <v-list-item @click="calendarType = 'day'">
                <v-list-item-title>Day</v-list-item-title>
              </v-list-item>
              <v-list-item @click="calendarType = 'week'">
                <v-list-item-title>Week</v-list-item-title>
              </v-list-item>
            </v-list>
          </v-menu>
        </v-toolbar>
      </v-sheet>

      <!-- Add event dialog -->
      <v-dialog v-model="addDialog" max-width="500">
        <v-card>
          <v-container>
            <v-form @submit.prevent="addEvent">
              <v-text-field v-model="content" type="text" label="Content"></v-text-field>
              <v-text-field v-model="detail" type="text" label="Detail"></v-text-field>
              <v-text-field v-model="date" type="date" label="Date"></v-text-field>
              <v-text-field v-model="time" format="24h" type="time" label="Time" id="addTimePicker"></v-text-field>
              <v-menu
                :offset-overflow="true"
                :offset-x="true"
                :open-on-click="true"
                :close-on-click="true"
                :close-on-content-click="false"
                activator="#addTimePicker"
              >
                <v-time-picker v-model="time" format="24hr"></v-time-picker>
              </v-menu>
              <v-text-field
                v-model="duration"
                format="24h"
                type="duration"
                label="Duration"
                id="addDurationPicker"
              ></v-text-field>
              <v-menu
                :offset-overflow="true"
                :offset-x="true"
                :open-on-click="true"
                :close-on-click="true"
                :close-on-content-click="false"
                activator="#addDurationPicker"
              >
                <v-time-picker v-model="duration" format="24hr"></v-time-picker>
              </v-menu>
              <v-combobox multiple v-model="tags" label="Tags" small-chips deletable-chips></v-combobox>
              <v-btn
                type="submit"
                color="primary"
                class="mr-4"
                @click.stop="dialog=false"
              >Create Note</v-btn>
            </v-form>
          </v-container>
        </v-card>
      </v-dialog>

      <v-sheet height="600">
        <v-calendar
          ref="calendar"
          v-model="focus"
          color="primary"
          :events="events"
          :event-color="getEventColor"
          :event-margin-bottom="3"
          :now="today"
          :type="calendarType"
          :interval-format="intervalFormat"
          @click:event="showEvent"
          @click:more="viewDay"
          @click:date="viewDay"
          @change="updateRange"
        ></v-calendar>
        <v-menu
          v-model="selectedOpen"
          :close-on-content-click="false"
          :activator="selectedElement"
          offset-x
        >
          <v-card color="grey lighten-4" min-width="350px" flat>
            <v-toolbar :color="selectedEvent.color" dark>
              <v-btn
                v-if="currentlyEditing !== selectedEvent.id"
                @click.prevent="editEvent(selectedEvent)"
                icon
              >
                <v-icon>mdi-pencil</v-icon>
              </v-btn>
              <v-btn v-else @click.prevent="closeEditEvent()" icon>
                <v-icon>mdi-close-outline</v-icon>
              </v-btn>
              <v-btn
                v-if="currentlyEditing === selectedEvent.id"
                @click.prevent="saveEvent(selectedEvent)"
                icon
              >
                <v-icon>mdi-check-outline</v-icon>
              </v-btn>
              <v-text-field
                v-if="currentlyEditing === selectedEvent.id"
                v-model="selectedEvent.content"
                hide-details
                single-line
              ></v-text-field>
              <v-toolbar-title v-else v-html="selectedEvent.content"></v-toolbar-title>
              <v-spacer></v-spacer>
              <!-- TODO: confirmation would be cool -->
              <v-btn @click="deleteEvent(selectedEvent.id)" icon>
                <v-icon>mdi-delete</v-icon>
              </v-btn>
              <!-- <v-btn icon> put delete into this
                <v-icon>mdi-dots-vertical</v-icon>
              </v-btn>-->
            </v-toolbar>
            <v-card-text>
              <form v-if="currentlyEditing !== selectedEvent.id">
                <div v-html="selectedEvent.detail"></div>
                <v-chip :key="tag.id" v-for="tag in selectedEvent.tags">
                  <div>{{ tag }}</div>
                </v-chip>
              </form>
              <form v-else>
                <TextareaAutosize
                  v-model="selectedEvent.detail"
                  type="text"
                  style="width: 100%"
                  :min-height="100"
                  placeholder="edit note..."
                ></TextareaAutosize>
                <v-combobox
                  multiple
                  v-model="selectedEvent.tags"
                  label="Tags"
                  small-chips
                  deletable-chips
                ></v-combobox>
                <v-text-field v-model="selectedEvent.date" type="date" label="Date"></v-text-field>
                <v-text-field
                  v-model="selectedEvent.time"
                  format="24h"
                  type="time"
                  label="Time"
                  id="editTimePicker"
                ></v-text-field>
                <v-menu
                  :offset-overflow="true"
                  :open-on-click="true"
                  :close-on-click="true"
                  :close-on-content-click="false"
                  activator="#editTimePicker"
                >
                  <v-time-picker v-model="selectedEvent.time" format="24hr"></v-time-picker>
                </v-menu>
                <v-text-field
                  v-model="selectedEvent.duration"
                  format="24h"
                  type="duration"
                  label="Duration"
                  id="editDurationPicker"
                ></v-text-field>
                <v-menu
                  :offset-overflow="true"
                  :open-on-click="true"
                  :close-on-click="true"
                  :close-on-content-click="false"
                  activator="#editDurationPicker"
                >
                  <v-time-picker v-model="selectedEvent.duration" format="24hr"></v-time-picker>
                </v-menu>
              </form>
            </v-card-text>
            <v-card-actions>
              <v-btn text color="secondary" @click="selectedOpen = false">Close</v-btn>
            </v-card-actions>
          </v-card>
        </v-menu>
      </v-sheet>
    </v-col>
  </v-row>
</template>

<script>
import axios from "axios";
import moment from "moment";

export default {
  data: () => ({
    today: moment()
      .format()
      .substring(0, 16),
    focus: null,
    calendarType: "day",
    typeToLabel: {
      week: "WEEK",
      day: "DAY"
    },
    content: null,
    detail: "",
    date: null,
    time: null,
    duration: null,
    tags: "",
    color: "#1976D2",
    rangeStart: null,
    rangeEnd: null,
    currentlyEditing: null,
    currentlyPickingTime: null,
    currentlyPickingDuration: null,
    selectedEvent: {},
    selectedElement: null,
    selectedOpen: false,
    events: [],
    tagsForEvents: [],
    addDialog: false,
    apiPassword: null
  }),
  computed: {
    title() {
      const start = this.rangeStart;
      const end = this.rangeEnd;
      if (!start || !end) {
        return "";
      }

      switch (this.calendarType) {
        case "week":
          return (
            moment(start.date).format("D.") +
            " - " +
            moment(end.date).format("D. MMM YYYY")
          );
        case "day":
          return moment(start.date).format("D. MMM YYYY");
        default:
          return "";
      }
    },
    monthFormatter() {
      return this.$refs.calendar.getFormatter({
        timeZone: "UTC",
        month: "long"
      });
    }
  },
  mounted() {
    this.getApiPassw();
    this.getEvents();
    this.focus = this.today;
  },
  methods: {
    async getTagsForEvents() {
      let res = await axios.get("http://notesbackend.k0haku.space/b/api/tags/");
      let tags = [];
      res.data.forEach(t => {
        let tag = {
          id: t.id,
          name: t.name,
          description: t.description,
          color: t.color
        };
        tags.push(tag);
      });
      this.tagsForEvents = tags;
    },
    async getEvents() {
      let res = await axios.get("http://notesbackend.k0haku.space/b/api/notes/");
      await this.getTagsForEvents();
      let events = [];
      res.data.forEach(n => {
        let durationArray = n.duration.split(":").map(Number);
        let allTags = [];
        this.tagsForEvents.forEach(t => {
          if (n.tags.includes(t.id)) {
            allTags.push(t.name);
          }
        });
        let note = {
          id: n.id,
          content: n.content,
          name: n.content,
          detail: n.detail,
          duration: n.duration.substring(0, n.duration.length - 3),
          start: moment
            .utc(n.time)
            .format()
            .substring(0, 16),
          end: moment
            .utc(n.time)
            .add(durationArray[0], "h")
            .add(durationArray[1], "m")
            .add(durationArray[2], "s")
            .format()
            .substring(0, 16),
          color: "#00aabb",
          tags: allTags
        };
        events.push(note);
      });
      this.events = events;
    },
    async deleteEvent(eventId) {
      await axios.delete(`http://notesbackend.k0haku.space/b/api/notes/${eventId}/`);
      this.selectedOpen = false;
      this.currentlyEditing = null;
      this.getEvents();
    },
    async saveEvent(event) {
      let newTagsIds = [];
      for (let i = 0; i < this.tagsForEvents.length; i++) {
        if (event.tags.some(testT => testT === this.tagsForEvents[i].name)) {
          newTagsIds.push(this.tagsForEvents[i].id);
        }
      }

      await axios.patch(`http://notesbackend.k0haku.space/b/api/notes/${event.id}/`, {
        content: event.content,
        detail: event.detail,
        tags: newTagsIds,
        duration: event.duration + ":00",
        time: event.date + "T" + event.time
      });
      this.selectedOpen = false;
      this.currentlyEditing = null;
      this.editingElement = null;
    },
    async addEvent() {
      if (this.content && this.time && this.date) {
        if (!this.duration) {
          this.duration = "00:00:00";
        }
        let newTagIds = [];
        const handleTags = async () => {
          for (let i = 0; i < this.tags.length; i++) {
            let t = this.tags[i];
            if (!this.tagsForEvents.some(testT => testT.name === t)) {
              let res = await axios.post(
                `http://notesbackend.k0haku.space/b/api/tags/`,
                {
                  name: t
                }
              );
              this.tagsForEvents.push({
                id: res.data.id,
                name: res.data.name,
                description: res.data.description,
                color: res.data.color
              });
            }
            for (let j = 0; j < this.tagsForEvents.length; j++) {
              if (this.tagsForEvents[j].name == t) {
                newTagIds.push(this.tagsForEvents[j].id);
              }
            }
          }
        };
        await handleTags();
        await axios.post(`http://notesbackend.k0haku.space/b/api/notes/`, {
          time: this.date + "T" + this.time,
          duration: this.duration,
          content: this.content,
          detail: this.detail,
          tags: newTagIds
        });
        this.getEvents();
        this.content = "";
        this.detail = "";
        this.time = "";
        this.date = "";
        this.duration = "";
        this.tags = "";
        this.addDialog = false;
      } else {
        alert("Name and start are required!");
      }
    },
    getApiPassw() {
      if (this.$cookies.isKey("cool-token")) {
        this.apiPassword = this.$cookies.get("cool-token");
        axios.defaults.headers.common["cool-token"] = this.apiPassword;
      }
    },
    setApiPassw() {
      axios.defaults.headers.common["cool-token"] = this.apiPassword;
      this.$cookies.set("cool-token", this.apiPassword);
    },
    getEventColor(event) {
      return event.color; // instead of using that, dependent on tag?
    },
    viewDay({ date }) {
      this.focus = date;
      this.calendarType = "day";
    },
    handleAddDialog() {
      this.addDialog = true;
      this.date = this.today.substring(0, 10);
      this.time = this.today.substring(11);
      this.duration = "00:00"
    },
    intervalFormat(timestamp) {
      return timestamp.time;
    },
    setToday() {
      this.focus = this.today;
    },
    prev() {
      this.$refs.calendar.prev();
    },
    next() {
      this.$refs.calendar.next();
    },
    editEvent(event) {
      this.currentlyEditing = event.id;
      this.selectedEvent.date = this.selectedEvent.start.substring(0, 10);
      this.selectedEvent.time = this.selectedEvent.start.substring(11);
    },
    closeEditEvent() {
      this.currentlyEditing = false;
    },
    showEvent({ nativeEvent, event }) {
      const open = () => {
        this.selectedEvent = event;
        this.selectedElement = nativeEvent.target;
        setTimeout(() => (this.selectedOpen = true), 10);
      };

      if (this.selectedOpen) {
        this.selectedOpen = false;
        this.currentlyEditing = null;
        setTimeout(open, 10);
      } else {
        open();
      }

      nativeEvent.stopPropagation();
    },
    updateRange({ start, end }) {
      // You could load events from an outside source (like database) now that we have the start and end dates on the calendar
      this.rangeStart = start;
      this.rangeEnd = end;
    },
    nth(d) {
      return d > 3 && d < 21
        ? "th"
        : ["th", "st", "nd", "rd", "th", "th", "th", "th", "th", "th"][d % 10];
    }
  }
};
</script>

<style lang="stylus" scoped></style>