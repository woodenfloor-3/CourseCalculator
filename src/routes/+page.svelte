<script>
    import { onMount } from 'svelte';
    import { db } from '$lib/firebase';
    import { collection, getDocs } from 'firebase/firestore';
  
    let courses = [];
    let selectedCourse = null;
    let studentJoinDate = '';
    let calculatedFees = null;
    let holidays = [];
  
    onMount(async () => {
      await fetchCourses();
      await fetchHolidays();
    });
  
    async function fetchCourses() {
      const querySnapshot = await getDocs(collection(db, 'courses'));
      courses = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      if (courses.length > 0) {
        selectedCourse = courses[0];
        studentJoinDate = selectedCourse.courseStartDate;
      }
    }
  
    async function fetchHolidays() {
      const querySnapshot = await getDocs(collection(db, 'holidays'));
      holidays = querySnapshot.docs.map(doc => doc.data());
    }
  
    function isHoliday(date) {
      const holiday = holidays.find(h => h.month === date.getMonth() + 1);
      return holiday ? holiday.days.includes(date.getDate()) : false;
    }
  
    function calculateFees() {
      if (!selectedCourse || !studentJoinDate) {
        alert('Please select a course and join date');
        return;
      }
  
      const courseStartDate = new Date(selectedCourse.courseStartDate);
      const joinDate = new Date(studentJoinDate);
      const courseEndDate = new Date(courseStartDate);
      courseEndDate.setMonth(courseEndDate.getMonth() + selectedCourse.durationMonths);
  
      if (joinDate < courseStartDate) {
        alert('Join date cannot be earlier than the course start date');
        return;
      }
  
      if (joinDate > courseEndDate) {
        alert('Join date is after the course has ended');
        return;
      }
  
      let currentDate = new Date(joinDate);
      let totalClasses = 0;
      let totalHours = 0;
  
      while (currentDate <= courseEndDate) {
        if (selectedCourse.classDays.includes(currentDate.getDay()) && !isHoliday(currentDate)) {
          totalClasses++;
          totalHours += selectedCourse.classHours;
        }
        currentDate.setDate(currentDate.getDate() + 1);
      }
  
      const totalFees = totalHours * selectedCourse.feePerHour;
      const taxRate = 0.1; // 10% tax rate
      const feesWithTax = totalFees * (1 + taxRate);
  
      calculatedFees = {
        course: selectedCourse.name,
        totalClasses,
        totalHours,
        totalFees,
        feesWithTax,
        courseStartDate: courseStartDate.toDateString(),
        studentJoinDate: joinDate.toDateString(),
        courseEndDate: courseEndDate.toDateString()
      };
    }
  </script>
  
  <main class="container mx-auto p-4 max-w-2xl">
    <h1 class="text-3xl font-bold mb-6 text-center text-blue-600">Course Fee Calculator</h1>
  
    <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
      <div class="mb-4">
        <label for="course" class="block text-gray-700 text-sm font-bold mb-2">Course:</label>
        <select id="course" bind:value={selectedCourse} class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
          {#each courses as course}
            <option value={course}>{course.name}</option>
          {/each}
        </select>
      </div>
  
      <div class="mb-4">
        <label for="courseStartDate" class="block text-gray-700 text-sm font-bold mb-2">Course Start Date:</label>
        <input type="date" id="courseStartDate" value={selectedCourse?.courseStartDate} readonly class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
      </div>
  
      <div class="mb-4">
        <label for="studentJoinDate" class="block text-gray-700 text-sm font-bold mb-2">Student Join Date:</label>
        <input type="date" id="studentJoinDate" bind:value={studentJoinDate} min={selectedCourse?.courseStartDate} class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
      </div>
  
      <div class="flex items-center justify-center">
        <button on:click={calculateFees} class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline">
          Calculate Fees
        </button>
      </div>
    </div>
  
    {#if calculatedFees}
      <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
        <h2 class="text-2xl font-bold mb-4 text-center text-blue-600">Calculation Results</h2>
        <div class="grid grid-cols-2 gap-4">
          <div class="font-bold">Selected Course:</div>
          <div>{calculatedFees.course}</div>
          <div class="font-bold">Course Start Date:</div>
          <div>{calculatedFees.courseStartDate}</div>
          <div class="font-bold">Student Join Date:</div>
          <div>{calculatedFees.studentJoinDate}</div>
          <div class="font-bold">Course End Date:</div>
          <div>{calculatedFees.courseEndDate}</div>
          <div class="font-bold">Total Classes:</div>
          <div>{calculatedFees.totalClasses}</div>
          <div class="font-bold">Total Hours:</div>
          <div>{calculatedFees.totalHours.toFixed(1)}</div>
          <div class="font-bold">Total Fees:</div>
          <div>¥{calculatedFees.totalFees.toFixed(2)}</div>
          <div class="font-bold">Fees with Tax:</div>
          <div>¥{calculatedFees.feesWithTax.toFixed(2)}</div>
        </div>
      </div>
    {/if}
  </main>
  
  <style>
    :global(body) {
      background-color: #f3f4f6;
    }
  </style>