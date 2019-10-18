<template>
  <v-app>
    <v-content>
      <DailyLog />
    </v-content>
  </v-app>
</template>

<script>
import DailyLog from "./components/DailyLog";

export default {
  name: "App",
  components: {
    DailyLog
  },
  data: () => ({
    //
  })
};
</script>
