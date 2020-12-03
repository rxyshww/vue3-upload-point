<template>
  <Upload />
</template>

<script>
import { defineComponent } from "vue";
import Upload from "./components/upload.vue";
export default defineComponent({
  name: "App",
  components: {
    Upload,
  },
});
</script>
