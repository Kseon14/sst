<template>
  <div class="shifts">
    <h3>Shifts in
      <select v-model="selectedMonth" v-on:change="getAll()">
        <option v-for="option in getMonthList(new Date().getFullYear())" v-bind:value="option.value">
          {{ option.text }}
        </option>
      </select>
    </h3>

    <div v-if="addAdjustment" class="modal-bg">
      <div class="modal-win">
        <h3>Add adjustment</h3>
        <div>
          <div class="clearfix">
            <label class="mdl-textfield_label" for="edit-item-lastName">Sum</label>&nbsp;
            <input class="mdl-textfield_input"  type="number" id="edit-item-lastName" v-model.number="newAdjustment.adjustment">
          </div>
          <div class="clearfix">
            <label class="mdl-textfield_label" for="edit-item-salary">Note</label>&nbsp;
            <input class="mdl-textfield_input" type="text" id="edit-item-salary" v-model="newAdjustment.adjustmentNote">
          </div>
        </div>
        <section>
          <br>
          <button v-on:click="addAdjustment = !addAdjustment; newAdjustment = {}">Close</button>&nbsp;
          <button id="new-item-edit" @click="createAdjustment" >Ok</button>
        </section>
        <p class="error">{{errorMessage}}</p>
      </div>
    </div>
    <div class="over">
      <div id="shiftTable" class="shift">
        <div class="shiftTh">
          <div class="shiftTd"></div>
          <div class="shiftTd" v-for="row in days">
            {{row.getDate()}}
          </div>
        </div>
        <template v-for="workerShifts in shifts">
          <div class="shiftTr">
            <div class="shiftTd"  v-bind:class="{ names: true }" v-on:click="$set(workerShifts, 'condition', !workerShifts.condition)">
              {{workerShifts.worker.lastName}} {{workerShifts.worker.firstName}}
            </div>
            <div class="shiftTd" v-for="day in days" v-on:click="clickMenu(day, workerShifts.worker.id)">
              <div v-if="!compareDate(day, workerShifts.shifts)"  v-bind:class="[getClassForTd(day, workerShifts, adjustments)]">
                <div v-bind:class="[getClass(workerShifts.worker.id, day)]">
                  <a href="#" v-on:click="addShift(day, workerShifts.worker.id)">+</a>
                  <a href="#" v-on:click="selectWorkerId(day, workerShifts.worker.id)">adjustment</a>
                </div>
              </div>
              <div v-else v-bind:class="[getClassForTd(day, workerShifts, adjustments)]">
              <div v-bind:class="[getClass(workerShifts.worker.id, day)]" v-if="!compareShiftState(day, workerShifts.shifts)">
                  <a href="#" v-on:click="rmShift(day, workerShifts.shifts, workerShifts.worker.id)">-</a>
                  <a href="#" v-on:click="closeShift(day, workerShifts.worker.id)">close shift</a>
                  <a v-if="!compareAdjustmentDates(day, adjustments[workerShifts.worker.id])"
                     href="#" v-on:click="selectWorkerId(day, workerShifts.worker.id)">adjustment</a>
              </div>
              </div>
            </div>
            <div class="shiftTd">{{workersSalaries[workerShifts.worker.id]}}</div>
          </div>
          <div class="shiftTr" v-if="workerShifts.condition && adj.adjustment" v-for="adj in adjustments[workerShifts.worker.id]" >
            <div class="shiftTd"><div >>> {{adj.adjustmentNote}}</div></div>
            <div class="shiftTd" v-for="day in days" v-if="compareAdjustmentDate(day, adj.adjustmentDate)"><div v-bind:class="[getClassForAdj(adj.adjustment)]" >{{adj.adjustment}}</div></div>
              <div class="shiftTd" v-else></div>
            <div class="shiftTd" v-if="!compareShiftState(new Date(adj.adjustmentDate), workerShifts.shifts)" v-on:click="rmAdjustment(adj.id, adj.worker.id)">−</div>
          </div>
        </template>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
  import {Component, Vue} from 'vue-property-decorator'
  import axios from 'axios';
  import {Worker} from './Worker.vue';

  export interface ShiftUser {
        worker: Worker;
        shifts: Shift[];
    }
    export class Shift{
        id!: number;
        worker!: Worker;
        shiftDate!: string;
        state!: string;
    }
    export class Salary{
        id?: number;
        worker?: Worker;
        salary?: number;
        salaryDate?: string;
    }

    export class Adjustment{
        id?: number;
        worker?: Worker;
        adjustment?: string;
        adjustmentDate?: string;
        adjustmentNote?: string;
    }

    export class TemporaryHolderMenuParams{
      day?: Date;
      workerShifts?: ShiftUser;
    }


    @Component
    export default class Shifts extends Vue {
        public shifts: ShiftUser[] = [];
        public selectedMonth: number = -1;
        public days: Date[] = [];
        public errorMessage:string = "";
        public workersSalaries: { [key: number]: any; } = {};
        public adjustments: { [key: number]: Adjustment[]; } = {};
        public addAdjustment : boolean = false;
        public newAdjustment : Adjustment = {};
        public tds: boolean [] = [];

        private async created() {
            await this.getAll();
        }

        private clickMenu(day: Date, id: any) {
            let key = Number(id +''+ day.getDate());

            for (let i = 0; i < this.tds.length; i++) {
                if (i != key) {
                    this.tds[i] = false;
                }
            }
            Vue.set(this.tds, key, !this.tds[key]);
        }

        private getClass(wokerId: number, day: Date) {
            let flag = this.tds[Number(wokerId +''+ day.getDate())];
            if (flag) {
                return "dropdownShow";
            } else {
                return "dropdown"
            }
        }

        private async getAll(){
            let month = localStorage.getItem("selectedMonth");
            console.log(month);
            if (month == null) {
                this.selectedMonth = this.getCurrentMonth();
                localStorage.setItem("selectedMonth", this.selectedMonth + "");
            } else if(this.selectedMonth == -1){
                this.selectedMonth = Number(month);
            } else if (this.selectedMonth != Number(month)) {
                localStorage.setItem("selectedMonth", this.selectedMonth + "");
            }

            this.getShifts().then(response => {
                this.shifts = response;
            });

            this.getSalaries().then(data => {
                console.log('work sal response', data);
                this.workersSalaries = data;
            });

            this.getAdjustments().then(data => {
                console.log('adj response', data);
                this.adjustments = data;
            });
        }

        private getCurrentMonth(){
            return new Date().getMonth() +1;
        }

        private getClassForTd(day : Date, workerShifts: ShiftUser, adjustments: any){
            let classes = "";
            if (this.isCurrentDay(day)) {
                classes = "currentDay "
            }
            if (this.compareShiftState(day, workerShifts.shifts)) {
                if (this.compareAdjustmentDates(day, adjustments[workerShifts.worker.id!])) {
                    return  classes + "gradientFilledPaid "
                } else {
                    classes = classes + "paidOut "
                }
            }
            if (this.compareAdjustmentDates(day, adjustments[workerShifts.worker.id!])) {
                if (this.compareDate(day, workerShifts.shifts)) {
                    classes = classes + "gradientFilled "
                } else {
                    classes = classes + "gradient "
                }
            } else {
                if (this.compareDate(day, workerShifts.shifts)) {
                    classes = classes + "filled "
                }
            }
            return classes;
        }

        private async getSalary(workerId : any) {
            let workerSalary: Salary[] = [];
            const response = await axios.get(
                '/api/salaries?worker=' + Number(workerId) + "&date=" + [new Date().getFullYear(), ("0" +this.selectedMonth).slice(-2), '01'].join('-'));
            workerSalary = response.data;
            this.workersSalaries[Number(workerId)] = workerSalary[0].salary;
        }

        private async getAdjustment(workerId : any) {
            const temp : { [key: number]: Adjustment[]; } = {};
            const response = await axios.get(
                '/api/salaries_adjustment?worker=' + workerId + '&date=' + [new Date().getFullYear(), ("0" + this.selectedMonth).slice(-2), '01'].join('-'));
            this.adjustments[workerId] = response.data;
        }

        private async getAdjustments() {
            const temp: { [key: number]: Adjustment[]; } = {};
            const response = await axios.get(
                '/api/salaries_adjustment?date=' + [new Date().getFullYear(), ("0" + this.selectedMonth).slice(-2), '01'].join('-'));
            let adjustments = response.data;
            let adjustment: Adjustment;
            for (adjustment of adjustments) {
                let array: Adjustment[] = [];
                let id: number = Number(adjustment.worker!.id);
                if (temp[id] == null) {
                    temp[id] = array;
                } else {
                    array = temp[id];
                }
                array.push(adjustment);
            }
            return temp;
        }

        private isPaid(entity : any){
            return entity.state == "PAID_OUT"
        }

        private async getSalaries(){
            const temp = {...this.workersSalaries};
            const response = await axios.get(
                '/api/salaries?date=' + [new Date().getFullYear(), ("0" +this.selectedMonth).slice(-2), '01'].join('-'));
            let workerSalaries = response.data;
            let  workerSalary = new Salary();
            for (workerSalary of workerSalaries) {
                let id : number = Number(workerSalary.worker!.id);
                if (workerSalary.salary == null) {
                    temp[id] = 0;
                    continue;
                }
                temp[id] = workerSalary.salary;
                for(let day of this.days) {
                    this.tds[(id*id + day.getDate())* id] = false;
                }
            }
            return temp;
        }

        private  getClassForAdj(adj : string) {
            if (Number(adj) < 0) {
                console.log( "red");
                return "commonTextRed";
            }
            console.log("green");
            return "commonTextGreen";
        }

        private async getShifts() {
            this.getDaysInMonth(this.selectedMonth -1, new Date().getFullYear())
            const response = await axios.get('/api/shifts/' +
                [new Date().getFullYear(), ("0" +this.selectedMonth).slice(-2), '01'].join('-'));
            return response.data;
        }

        private compareDate(day: Date, shifts: Shift[]) {
            if (shifts && shifts.length && shifts[0].shiftDate === null) {
                return false;
            }
            for (let shift of shifts) {
                if (day.getDate() == new Date(shift.shiftDate).getDate()) {
                    return true;
                }
            }
            return false
        }

        private compareAdjustmentDate(day: Date, date: string) {
            if (date === null) {
                return false;
            }
            if (day.getDate() == new Date(date).getDate()) {
                return true;
            }
            return false
        }

        private compareAdjustmentDates(day: Date, adjustments: any) {
            if (!adjustments || !adjustments.length) {
                return false;
            }
            for(let adj of adjustments) {
                if (!adj) {
                    return;
                }
                if (adj.adjustmentDate === null) {
                    continue;
                }
                if (day.getDate() == new Date(adj.adjustmentDate).getDate()) {
                    return true;
                }
            }
            return false
        }

        private compareShiftState(day: Date, shifts: Shift[]){
            if (shifts && shifts.length && shifts[0].shiftDate === null) {
                return false;
            }
            for (let shift of shifts) {
                if (day.getDate() == new Date(shift.shiftDate).getDate() && shift.state == 'PAID_OUT') {
                    return true;
                }
            }
            return false;
        }

        private isCurrentDay(day: Date) {
            return new Date().getDate() == day.getDate() && new Date().getMonth() == day.getMonth();
        }

        private getMonthList(year: number) {
            let i;
            let months = [];
            for (i = 0; i < 12; i++) {
                months.push(new Option(new Date(year, i, 1).toLocaleString('en-us', {month: 'long'}), i + 1 + ''))
            }
            return months;
        }

        private async createAdjustment(){
            let adjustment = this.newAdjustment;
            let worker = adjustment.worker!;
            await axios.post('/api/salaries_adjustment', adjustment)
                .then(() => {
                        Promise.all([this.getShifts(), this.getSalary(worker.id!), this.getAdjustment(worker.id!)]).then(data =>
                            this.shifts = data[0]);
                    }
                ).catch(error => {
                    this.errorMessage = error.response.data.message
                });
            this.addAdjustment = !this.addAdjustment;
            this.newAdjustment = {};
        }

        private async addShift(day : Date, workerId: number) {
            let shift = new Shift();
            console.log(day.getMonth());

            shift.shiftDate = day.getFullYear() + "-" + ("0" + (day.getMonth()+1)).slice(-2) + "-" + ("0" + day.getDate()).slice(-2);
            let worker: Worker = {};
            worker.id = workerId;
            shift.worker = worker;
            await axios.post('/api/shifts', shift)
                .then(() => {
                        Promise.all([this.getShifts(), this.getSalary(worker.id)]).then(data =>
                            this.shifts = data[0]);
                    }
                ).catch(error => {
                    this.errorMessage = error.response.data.message
                });
            console.log(workerId);
        }

        private async rmAdjustment(adjustmentId: number,  workerId: number) {
            let worker: Worker = {};
            let shift =  new Shift();
            worker.id = workerId;
            shift.worker = worker;
            await axios.delete('/api/salaries_adjustment/' + adjustmentId)
                .then(() => {
                        Promise.all([this.getShifts(), this.getSalary(workerId), this.getAdjustments()]).then(data =>
                            this.shifts = data[0]);
                    }
                ).catch(error => {
                    this.errorMessage = error.response.data.message
                });
            console.log(workerId);
            this.getAdjustment(workerId)
        }

        private async rmShift(day: Date, shifts: Shift[], workerId: number) {
            let shift = new Shift();
            for (shift of shifts) {
                if (day.getDate() == new Date(shift.shiftDate).getDate()) {
                    break;
                }
            }
            await axios.delete('/api/shifts/' + shift.id)
                .then(() => {
                        Promise.all([this.getShifts(), this.getSalary(workerId)]).then(data =>
                            this.shifts = data[0]);
                    }
                ).catch(error => {
                    this.errorMessage = error.response.data.message
                });
            console.log(workerId);
        }

        private getDaysInMonth(month: number, year: number) {
            // Since no month has fewer than 28 days
            var date = new Date(year, month, 1);
            this.days = [];
            while (date.getMonth() === month) {
                this.days.push(new Date(date));
                date.setDate(date.getDate() + 1);
            }
        }
        private selectWorkerId(day :Date, workerId: number){
            this.addAdjustment = !this.addAdjustment;
            console.log(workerId);
            let worker: Worker = {};
            worker.id = workerId;
            this.newAdjustment.worker = worker;
            this.newAdjustment.adjustmentDate = new Date().getFullYear() + "-" + ("0" + (this.selectedMonth  )).slice(-2)
                + "-" + ("0" + day.getDate()).slice(-2);
        }

        private async closeShift(day :Date, workerId: number){
            let shift = new Shift();
            console.log(day.getMonth());
            shift.shiftDate = day.getFullYear() + "-" + ("0" + (day.getMonth()+1)).slice(-2) + "-" + ("0" + day.getDate()).slice(-2);
            let worker: Worker = {};
            worker.id = workerId;
            shift.worker = worker;
            await axios.post('/api/shifts/closeShift', shift)
                .then(() => {
                        Promise.all([this.getShifts(), this.getSalary(worker.id)]).then(data =>
                            this.shifts = data[0]);
                    }
                ).catch(error => {
                    this.errorMessage = error.response.data.message
                });
            console.log(workerId);
        }
    }

</script>
<style>
  .shift {
    font-family: 'Open Sans', sans-serif;
    /*width: 60%;*/
    border-collapse: collapse;
    border-spacing: 0;
    margin-left:auto;
    margin-right:auto;
    display: table;
    /*display: block;*/
  }

  .shiftTh {
    display: table-row;
    padding: 5px;
    min-width: 10px;
    text-align: center;
  }

  .shiftTr:hover {
    background-color: #f5f5f5;
  }

  .shiftTr {
    display: table-row;
  }

  .shiftTd {
    display: table-cell;
    position: relative;
    border-bottom: 1px solid rgba(170, 179, 232, 0.17);
    border-top: 1px solid rgba(170, 179, 232, 0.17);
    border-right: 1px solid rgba(170, 179, 232, 0.17);
    text-align: center;
    cursor: default;
    min-width: 25px;
    min-height: 30px;
  }

  .shiftTd:hover {
    background-color: #42b983;
  }

  .shiftTd.names {
    min-width: 150px;
    text-align: left;
  }

  .modal-bg {
    /*background-color: rgba(0,0,0, 0.5);*/
    position: fixed;
    top: 25%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 2;
  }

  .over{
    overflow: auto;
    height: 300px;
  }

  .modal-win {
    border: 1px lightslategrey solid;
    background-color: white;
    border-radius: .25rem;
    padding: 20px;
    width: 300px;
    max-width: 100%;
  }

  .mdl-textfield_input {
    border: 0;
    border-bottom: 1px solid rgba(170, 179, 232, 0.17);
    padding: 5px;
    display: inline-block;
    float: left;
    text-align: left;
  }

  .mdl-textfield_label {
    border: 0;
    min-width: 12px;
    display: inline-block;
    padding: 5px;
    float: left;
    clear: left;
    width: 100px;
    text-align: left;
  }

  input:focus {
    outline: none;
    border: 0;
    border-bottom: 1px solid rgba(170, 179, 232, 0.17);
    min-width: 20px;
    padding: 5px;
  }

  button {
    align-self: center;
    background-color: white;
    border-radius: .25rem;
    color: black;
    border: 1px solid #42b983;
    padding: 5px 15px;
    text-align: center;
    font-size: 13px;
    text-decoration: none;
    display: inline-block;
    transition-duration: 0.4s;
  }

  button:hover {
    background-color: #42b983;
    color: white;
  }

  .clearfix:before {
    display: table;
    content: "";
  }

  .clearfix:after {
    display: table;
    clear:both;
    content: "";
  }

  .commonTextRed {
    text-align: center;
    color: #dd1144;
    font-weight: bold;
  }
  .commonTextGreen {
    text-align: center;
    color: #00b72d;
    font-weight: bold;
  }

  .currentDay {
    background-color: #ff6500;
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
  }

  .filled {
    background-color: #206600;
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
  }
  .bigSign {
    font-size: 120%;
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
  }

  .gradientFilled {
    background: linear-gradient(to right bottom, #206600 50%, #42b983 50%);
    color: white;
    border: none;

    height:100%;
    position: absolute;
    width: 100%;
    left: 0;
    top: 0;
  }

  .gradientFilledPaid {
    background: linear-gradient(to right bottom, #fff400 50%, #42b983 50%);
    color: white;
    border: none;

    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
  }

  .gradient {
    background: linear-gradient(to right bottom, #ffffff 50%, #42b983 50%);
    color: black;
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
  }

  .dropdown {
    display: none;
    position: absolute;
    background-color: #f1f1f1;
    min-width: 90px;
    z-index: 1;
  }


  .dropdown a {
    color: black;
    padding: 3px;
    text-decoration: none;
    display: block;
  }

  .dropdownShow {
    display: block;
    position: absolute;
    background-color: #f1f1f1;
    min-width: 90px;
    left: 10px;
    top: 10px;
    z-index: 1;
  }

  .dropdownShow a {
    color: black;
    padding: 3px;
    text-decoration: none;
    display: block;
  }

  .paidOut {
    background-color: #fff400;
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
  }

</style>
