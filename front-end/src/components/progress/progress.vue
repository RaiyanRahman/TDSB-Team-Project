<template>
  <div class="col-12" id="container" style="margin-top: 5%;">
    <navbar></navbar>
    <div class="col-4" style="float: right; padding-top: 15px;">
      <div class="card" style="border:1px solid black; margin-bottom:20px">
        <div class="card-header">
          <h6 style="float: left; padding-top: 10px">Courses</h6>
          <div class="form-group">
            <select class="form-control" v-model="selectedCourse">
              <option value="" selected disabled hidden>Pick a Course</option>
              <option v-for="course in courseNames" :key="course">{{course}}</option>
            </select>
          </div>
          <div v-if="selectedCourse">
            <b-modal
              id="modal-1"
              size="lg"
              centered
              ref="modal"
              title="Update Progress"
              @ok="updateProgress"
            >
              <label for="completedAssignment">Amount Completed</label>
              <div class="form-group">
                <vue-slider v-model="progressValue" :marks="marks">
                  <template v-slot:step="{ label, active }">
                    <div :class="['custom-step', { active }]"></div>
                  </template>
                </vue-slider>
              </div>
            </b-modal>
            <hr>
            <b-tabs content-class="mt-3" fill pills>
              <b-tab title="Progress" active>
                <b-card style="max-height: 505px; overflow-y: scroll;">
                  <b-list-group flush>
                    <b-list-group-item v-for="assignment in assignmentList" :key="assignment.title">
                      <p style="float: left">{{assignment.title}}</p>
                      <b-button v-on:click="deleteAssignment(assignment)" size="sm" variant="danger" style="float: right; margin-top: -5px; margin-left: 10px; border-radius: 15px;">
                        <font-awesome-icon :icon="['fas', 'trash']" />
                      </b-button>
                      <b-button v-b-modal.modal-1 variant="info" v-on:click="editingAssignment = assignment; progressValue = assignment.progress" size="sm" style="float: right; margin-top: -5px; border-radius: 15px;">
                        <font-awesome-icon :icon="['fas', 'edit']" />
                      </b-button>
                    </b-list-group-item>
                  </b-list-group>
                </b-card>
              </b-tab>
              <b-tab title="Grades">
                <b-card style="max-height: 505px; overflow-y: scroll;">
                  <b-list-group flush>
                    <b-list-group-item v-for="assignment in assignmentList" :key="assignment.title">
                      <p style="float: left">{{assignment.title}}</p>
                      <b-form-input :state="nameState(assignment.grade)" v-on:blur="updateGrade(assignment)" :type="'number'" id="input-grade" size="sm" v-model="assignment.grade" style="float: right; width: 30%" placeholder="Grade"></b-form-input>
                    </b-list-group-item>
                    <b-list-group-item>
                      <h5>Current Average:  {{getAverage(assignmentList)}}</h5>
                    </b-list-group-item>
                  </b-list-group>
                </b-card>
              </b-tab>
            </b-tabs>
          </div>
        </div>
      </div>
    </div>

    <div class="col-8" style="padding-top: 15px">
      <div class="card" style="border:1px solid black;">
        <div class="card-header">
          <h6 style="float: left; padding-top: 10px">Grades over Time</h6>
        </div>
        <div>
          <TrendChart
            :datasets="chartData"
            :grid="{
              verticalLines: true,
              horizontalLines: true
            }"
            :labels="{
              // xLabels: xAxisList,
              yLabels: 2
            }"
            :min="0"
            :max="100"
            :interactive="true">
          </TrendChart>
        </div>
      </div>
    </div>

    <div class="col-8" style="padding-top: 15px">
      <div class="card" style="border:1px solid black; padding-bottom:10px">
        <div class="card-header" style="margin-bottom:20px">
          <h6 style="float: left; padding-top: 10px">Completion Per Assignment</h6>
        </div>
        <div>
          <pure-vue-chart v-if="selectedCourse"
            :points="barGraphData"
            :show-y-axis="true"
            :show-x-axis="true"
            :show-values="true"
            :show-trend-line="false"
            :width="800"
            :height="375"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import navbar from "../navbar/navbar";
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/antd.css";
import PureVueChart from "pure-vue-chart";
import TrendChart from "vue-trend-chart";

export default {
  name: "Progress", //this is the name of the component
  components: {
    VueSlider,
    PureVueChart,
    TrendChart,
    navbar: navbar
  },
  data() {
    return {
      user: null,
      progressValue: 50,
      editingAssignment: null,
      courses: [],
      selectedCourse: "",
      selectedCourseData: {},
      courseNames: [],
      assignmentList: [],
      chartData: [
              {
                data: [],
                smooth: true,
                fill: false
              }
            ],
      xAxisList: [],
      barGraphData: [],
      marks: val => val % 20 === 0
    };
  },
  methods: {
    showMe: function(obj) {
      console.log(obj) // eslint-disable-line no-console
    },
    getAverage: function(assignmentList) {
      // Compute average
      var length = assignmentList.length
      var sum = 0
      for (var i = 0; i < length; i++) {
        sum += parseInt(assignmentList[i].grade)
      }
      var avg = sum / length
      return avg.toFixed(2)
    },
    deleteAssignment: function(assignment) {
      for (var i = 0; i < this.assignmentList.length; i++) {
        if (assignment.title === this.assignmentList[i].title) {
          this.assignmentList.splice(i, 1)
          break
        }
      }
      this.getChartData()
      this.getGraphData()
    },
    getCourseInformation: function(courseName) {
      // Get the course dictionary.
      for (var coursei = 0; coursei < this.courses.length; coursei++) {
        if (this.courses[coursei].name === courseName) {
          this.selectedCourseData = this.courses[coursei]
        }
      }
      // Get the course's assignments from the backend.
      // Set all the course information.
      this.$axios
        .get(this.$store.state.prefix + "/api/task/student/course?student=" + this.user._id + "&course=" + this.selectedCourseData.id)
        .then(response => {
          this.assignmentList = response.data
          // If the grade and progress are not defined, define them.
          for (var a = 0; a < this.assignmentList.length; a++) {
            if (!('progress' in this.assignmentList[a])) {
              this.assignmentList[a].progress = 0
            }
            if (!('grade' in this.assignmentList[a])) {
              this.assignmentList[a].grade = 0
            }
          }
          // Set the bar graph and chart data.
          this.getGraphData()
          this.getChartData()
        });
    },
    getCompletionPercentage: function(assignmentList) {
      // Convert the assignments to their completion percentage.
      return assignmentList
    },
    nameState: function(grade) {
      return grade >= 0 && grade <= 100 ? true : false
    },
    getCourses: function() {
      this.courseNames = []
      this.user = this.$store.state.user
      this.courses = this.user.courses
      for (var i = 0; i < this.courses.length; i++) {
        this.courseNames.push(this.courses[i].name)
      }
    },
    getChartData: function() {
      var myData = []
      this.xAxisList = []
      for (var i = 0; i < this.assignmentList.length; i++) {
        myData.push(parseInt(this.assignmentList[i].grade))
        this.xAxisList.push(this.assignmentList[i].title)
      }
      this.chartData = [{
        data: myData,
        smooth: true,
        fill: true
      }]
    },
    getGraphData: function() {
      this.barGraphData = []
      var data = []
      for (var i = 0; i < this.assignmentList.length; i++) {
        // var data = {label: this.assignmentList[i].title, value: this.assignmentList[i].progress}
        // this.barGraphData.push(data)
        var num = parseInt(this.assignmentList[i].progress)
        // var name = this.assignmentList[i].title
        // data.push({label: name, value: num})
        data.push(num)
      }
      this.barGraphData = data
    },
    updateProgress: function() {
      this.editingAssignment.progress = this.progressValue
      console.log("setting " + this.editingAssignment.title + "'s progress to " + this.progressValue)  // eslint-disable-line no-console
      var datetime = this.separateDateTime(this.editingAssignment.deadline)
      var updateData = {
        title: this.editingAssignment.title,
        course_id: this.editingAssignment.course_id,
        description: this.editingAssignment.description,
        attachments: this.editingAssignment.attachments,
        grade: this.editingAssignment.grade,
        progress: this.progressValue,
        date: datetime.date,
        time: datetime.time
      }
      this.$axios
        .patch(this.$store.state.prefix + "/api/task/" + this.editingAssignment._id, updateData)
        .then(response => {
          console.log(response.data) // eslint-disable-line no-console
        });
      for (var i = 0; i < this.assignmentList.length; i++) {
        if (this.assignmentList[i].title === updateData.title) {
          this.assignmentList[i].progress = this.progressValue
        }
      }
      this.getGraphData()
    },
    updateGrade: function(assignment) {
      var datetime = this.separateDateTime(assignment.deadline)
      var updateData = {
        title: assignment.title,
        course_id: assignment.course_id,
        description: assignment.description,
        attachments: assignment.attachments,
        grade: assignment.grade,
        progress: assignment.progress,
        date: datetime.date,
        time: datetime.time
      }
      this.$axios
        .patch(this.$store.state.prefix + "/api/task/" + assignment._id, updateData)
        .then(response => {
          console.log(response)   // eslint-disable-line no-console
        });
      this.getChartData()
    },
    separateDateTime: function(datetime) {
      var monthDict = {1: "Jan", 2: "Feb", 3: "Mar",
                    4: "Apr", 5: "May", 6: "June",
                    7: "July", 8: "Aug", 9: "Sept",
                    10: "Oct", 11: "Nov", 12: "Dec"}
      datetime = datetime.split("T")
      var date = datetime[0].split("-").reverse()
      date[1] = monthDict[parseInt(date[1])]
      date = date.join(" ")
      var time = datetime[1].substring(0, datetime[1].length - 3)
      var ret_val = {date: date, time: time}
      return ret_val
    }
  },
  watch: {
    selectedCourse: function() {
      this.getCourseInformation(this.selectedCourse)
    }
  },
  created: function() {
    this.getCourses()
  }
};
</script>
<style>
.custom-step {
  width: 100%;
  height: 90%;
  border-radius: 50%;
  box-shadow: 0 0 0 3px #ccc;
  background-color: #fff;
}
.custom-step.active {
  box-shadow: 0 0 0 3px #3498db;
  background-color: #3498db;
}
</style>