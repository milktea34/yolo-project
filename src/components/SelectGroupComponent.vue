<template>
  <select v-for="(group, groupIdx) in this.groupList" :disabled="group.disabled"
          v-on:change="onChangeSelect($event, groupIdx)"
          v-bind:key="groupIdx" v-model="this.selectedOptions[groupIdx]">
    <option value="">{{ group.title }}</option>
    <option v-for="(option, optionIdx) in group.options" v-bind:key="optionIdx" :value="option.name"
            :disabled="option.count === 0">
      {{ option.name }}
      {{ option.count === 0 ? "(품절)" : groupIdx === this.groupList.length - 1 ? "(" + option.count + "개 구입가능)" : "" }}
    </option>
  </select>
  <div class="selected-option-name" v-if="this.selectedOptionsName">
    {{ this.selectedOptionsName }}
  </div>
</template>
<script>
import axios from "axios";

export default {
  name: 'SelectGroupComponent',
  data() {
    return {
      groupList: [],
      countList: [],
      selectedOptions: {},
      lastChangeIdx: 0
    }
  },
  computed: {
    selectedOptionsName() {
      const selectOptions = Object.values(this.selectedOptions).filter((option) => {
        return option;
      });
      return selectOptions.length === this.groupList.length ? selectOptions.join('/') : '';
    }
  },
  created() {
    axios.get("https://apigen-server.herokuapp.com/api/621f2588d8d85b8d78eb3e64").then((response) => {
      console.log(response.data.data);
      this.groupList = response.data.data.groupList;
      this.countList = response.data.data.countList;
      this.groupList.forEach((group, idx) => {
        this.selectedOptions[idx] = '';
        group.disabled = (idx !== 0); // 첫번째꺼 빼고 전부 비활성화
        group.options.forEach((option, optIdx) => {
          group.options[optIdx] = {name: option, count: this._calcRemainCount([option], this.countList)}
        })
      })
    })
  },
  methods: {
    _calcRemainCount(selectedValArr, countList) {
      const intersect = countList.filter((count) => {
        const combArr = Object.values(count.combination);
        const isIncluded = selectedValArr.every((option) => {
          return combArr.indexOf(option) > -1;
        });
        return isIncluded;
      })
      const remainCount = intersect.map((value) => {
        return value.remainCount;
      }).reduce((prev, next) => prev + next, 0);
      return remainCount;
    },

    onChangeSelect(event, groupIdx) {
      this.selectedOptions[groupIdx] = event.currentTarget.value;
      const nextIdx = groupIdx + 1;
      const isChangedPlus = this.lastChangeIdx <= groupIdx;
      this.groupList.forEach((group, i) => {
        if (!isChangedPlus) {
          this.groupList[i].disabled = !(i < nextIdx);
          if (i >= nextIdx) {
            this.selectedOptions[i] = '';
          }
        } else {
          this.groupList[i].disabled = !(i <= nextIdx);
        }
      });
      this.groupList.forEach((group, groupIdx) => {
        const target = Object.values(this.selectedOptions).slice(0, groupIdx).filter((option) => {
          return option
        });
        group.options.forEach((option, optionIdx) => {
          group.options[optionIdx].count = this._calcRemainCount([...target, option.name], this.countList);
        });
      })
      this.lastChangeIdx = groupIdx;
    }
  }
}
</script>
<style>

.selected-option-name {
  background-color: lightskyblue;
  padding: 0.25rem;
  font-weight: bold;
}

select {
  display: flex;
  width: 100%;
  padding: 0.5rem;
  margin: 0.5rem 0rem;
}
</style>