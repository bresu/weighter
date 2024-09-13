<template>
  <div id="app">
    <h1>Zeugnis Hochladen</h1>
    <input type="file" @change="handleFileUpload" accept="application/pdf" />

    <!-- Display the GPA prominently -->
    <div v-if="gpa !== null">
      <h2>Gewichteter Notendurchschnitt (GPA): {{ gpa }}</h2>
      <p>Total ECTS: {{ totalECTS }}</p>
    </div>

    <!-- Display the parsed courses in a table -->
    <table v-if="Object.keys(courses).length > 0" border="1" cellpadding="10" cellspacing="0">
      <thead>
        <tr>
          <th>Kursname</th>
          <th>ECTS</th>
          <th>Note</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(course, name) in courses" :key="name">
          <td>{{ name }}</td>
          <td>{{ course.ects }}</td>
          <td>{{ course.grade }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import * as pdfjsLib from "pdfjs-dist/legacy/build/pdf"; // Import PDF.js
import "pdfjs-dist/legacy/build/pdf.worker.entry"; // Import PDF.js Worker

export default {
  data() {
    return {
      pdfContent: null,
      courses: {}, // To store the parsed courses
      gpa: null,   // To store the final GPA
      totalECTS: null // To store the total ECTS
    };
  },
  methods: {
    parseTranscript(text) {
      const courses = {};
      const COURSE_REGEX = /(.+?)\s+(\d+(?:\.\d+)?)\s+ECTS\s+.*?Note:\s*(\d|\+)/g;
      
      let match;
      while ((match = COURSE_REGEX.exec(text)) !== null) {
        const courseName = match[1].trim(); // Capture course name
        const ects = parseFloat(match[2]);  // Capture ECTS value
        const grade = match[3].trim();      // Capture grade (1-5 or "+")
        
        // Ignore invalid grades
        if (!/^[1-4]$/.test(grade)) {
          continue;
        }

        courses[courseName] = { ects, grade: parseInt(grade, 10) };
      }

      return courses;
    },

    calculateGPA(courses, includeNegativeGrades = false) {
      let totalECTS = 0;
      let weightedECTSGradeSum = 0;

      Object.values(courses).forEach(({ ects, grade }) => {
        if (!includeNegativeGrades && grade === 5) {
          return; // Skip negative grades
        }
        totalECTS += ects;
        weightedECTSGradeSum += ects * grade;
      });

      if (totalECTS === 0) {
        return { gpa: "N/A", totalECTS }; // Avoid division by zero
      }

      const gpa = weightedECTSGradeSum / totalECTS;
      return { gpa: gpa.toFixed(2), totalECTS };
    },

    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = async (e) => {
          const typedArray = new Uint8Array(e.target.result);
          const pdf = await pdfjsLib.getDocument(typedArray).promise;
          let parsedText = "";

          for (let i = 1; i <= pdf.numPages; i++) {
            const page = await pdf.getPage(i);
            const textContent = await page.getTextContent();
            const pageText = textContent.items.map((item) => item.str).join(" ");
            parsedText += pageText + "\n";
          }

          // Parse the extracted text
          this.courses = this.parseTranscript(parsedText);

          // Calculate GPA
          const { gpa, totalECTS } = this.calculateGPA(this.courses);
          this.gpa = gpa;
          this.totalECTS = totalECTS;

          console.log("Parsed Courses:", this.courses);
          console.log("GPA (without negative grades):", gpa, "Total ECTS:", totalECTS);
        };
        reader.readAsArrayBuffer(file);
      }
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  margin-top: 50px;
}

input {
  margin-bottom: 20px;
}

table {
  width: 80%;
  margin: 20px auto;
  border-collapse: collapse;
}

th, td {
  padding: 10px;
  text-align: left;
}

h2 {
  color: #333;
  font-size: 24px;
  margin: 20px 0;
}
</style>
