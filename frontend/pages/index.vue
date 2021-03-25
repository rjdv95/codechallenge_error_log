<template>
  <div class="text-gray-800 text-base font-bold">
    <header>
      <nav class="flex flex-row-reverse justify-between h-3 pt-3 pr-3 pb-0">
        <div>
          <button
            class="w-20 text-white text-base font-bold bg-blue-500 hover:bg-blue-700 p-2 rounded-md"
            v-on:click="undo"
          >undo</button>
          <button
            class="w-20 text-white text-base font-bold bg-red-500 hover:bg-red-700 p-2 rounded-md"
            v-on:click="reset"
          >reset</button>
          <button
            class="w-20 text-white text-base font-bold bg-green-500 hover:bg-green-700 p-2 rounded-md"
            v-on:click="submitResulved"
          >submit</button>
        </div>
      </nav>
    </header>

    <div
      class="flash-message flex fixed top-0 w-full justify-center items-center flex-wrap h-12 shadow-xl px-2 my-2 mx-2 bg-green-300"
      v-if="showMessage"
    >{{ message }}</div>
    <div class="flex flex-col mt-8">
      <!------------unresolved--------->

      <div class="flex w-full justify-center items-center bg-blue-200 py-4 mt-4 h-12">
        <h3>Unresolved</h3>
      </div>
      <div class="FixedHeightContainer overflow-auto h-40">
        <div
          v-for="issue in unresolved"
          class="flex justify-between items-center flex-wrap h-10 shadow-md px-2 my-2 mx-2 bg-gray-200 hover:bg-gray-400"
          :key="issue.id"
        >
          <div>
            <span class="space-x-3">{{ issue.code }}</span>
            <span>{{ issue.text }}</span>
          </div>
          <button
            class="w-20 text-white bg-blue-500 hover:bg-blue-700 p-2"
            v-on:click="resolve(issue)"
          >resolve</button>
        </div>
      </div>
      <!-----------resolved------>
      <div class="flex w-full justify-center items-center bg-blue-200 py-4 mt-4 h-12">
        <h3>Resolved</h3>
      </div>
      <div class="FixedHeightContainer overflow-auto h-40">
        <div
          class="flex justify-between items-center flex-wrap h-10 shadow-md px-2 my-2 mx-2 bg-gray-200 hover:bg-gray-400"
          v-for="issue in resolved"
          :key="issue.id"
        >
          <div>
            <span class="space-x-3">{{ issue.code }}</span>
            <span>{{ issue.text }}</span>
          </div>
          <button
            class="w-20 text-white bg-blue-500 hover:bg-blue-700 p-2"
            v-on:click="unresolve(issue)"
          >unresolve</button>
        </div>
      </div>

      <!----------backlog------->

      <div class="flex w-full justify-center items-center bg-blue-200 py-4 mt-4 h-12">
        <h3>Backlog</h3>
      </div>
      <div class="FixedHeightContainer overflow-auto h-40">
        <div
          class="flex justify-between items-center flex-wrap h-10 shadow-md px-2 my-2 mx-2 bg-gray-200 hover:bg-gray-400"
          v-for="issue in backlog"
          :key="issue.id"
        >
          <div>
            <span class="space-x-3">{{ issue.code }}</span>
            <span>{{ issue.text }}</span>
          </div>
          <button
            class="w-20 text-white bg-blue-500 hover:bg-blue-700 p-2"
            v-on:click="move(issue)"
          >move</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
// storing issues/errors for reseting when needed
var gStore = {};
export default {
  async asyncData({ $axios }) {
    try {
      const { resolved, unresolved, backlog } = await $axios.$get(
        "http://localhost:8000/get_lists/"
      );
      gStore = {
        resolved: [...resolved],
        unresolved: [...unresolved],
        backlog: [...backlog]
      };
      const store = gStore;
      return {
        resolved,
        unresolved,
        backlog,
        store
      };
    } catch (error) {
      console.log(
        `Couldn't get error lists:\n${error}\nDid you start the API?`
      );
    }
  },
  methods: {
    // exchange item from first list to another and saving data for undo
    replace: function(item, array1, array2) {
      this.changedItem = item;
      this.changedIndex = array1.indexOf(item);
      this.fromArry = array2;
      this.toArry = array1;
      array1.splice(array1.indexOf(item), 1);
      array2.push(item);
    },
    flashMessage: function(message) {
      this.message = message;
      this.showMessage = true;
      setTimeout(() => {
        this.showMessage = false;
      }, this.messageTime);
    },
    unresolve: function(issue) {
      this.replace(issue, this.resolved, this.unresolved);
    },
    resolve: function(issue) {
      this.replace(issue, this.unresolved, this.resolved);
    },
    // move backlog item to unresolved
    move: function(issue) {
      this.replace(issue, this.backlog, this.unresolved);
    },
    undo: function() {
      if (this.changedItem) {
        this.fromArry.splice(this.fromArry.indexOf(this.changedItem), 1);
        this.toArry.splice(this.changedIndex, 0, this.changedItem);
        let message = `issue with code: ${
          this.changedItem.code
        } undone successfully`;
        this.flashMessage(message);
        this.changedItem = this.fromArry = this.toArry = this.changedIndex = null;
      }
    },
    // undo-all/reset to original values
    reset: function() {
      this.resolved = this.store.resolved;
      this.unresolved = this.store.unresolved;
      this.backlog = this.store.backlog;
      let message = `reseted successfully`;
      this.flashMessage(message);
    },
    submitResulved: async function() {
      const data = {
        resolved: this.resolved
      };
      const res = await this.$axios({
        method: "put",
        url: "http://localhost:8000/put_resolved",
        headers: {
          "Content-Type": "application/json"
          // "Access-Control-Allow-Origin": "*"
        },
        data: data
      })
        .then(() => {
          let message = `all resolved issues submitted successfully`;
          this.flashMessage(message);
        })
        .catch(e => console.log(e));
    }
  },
  data() {
    return {
      resolved: [],
      unresolved: [],
      backlog: [],
      store: {},
      changedItem: null,
      changedIndex: null,
      fromArry: null,
      toArry: null,
      reseted: false,
      undone: false,
      submitted: false,
      showMessage: false,
      message: "",
      messageTime: 1500
    };
  }
};
</script>