<script>
  import { onMount } from 'svelte';
  import { db } from '$lib/firebase';
  import { collection, getDocs } from 'firebase/firestore';

  let categories = [];
  let selectedCategory = null;
  let courses = [];
  let filteredCourses = [];
  let selectedCourse = null;
  let studentJoinDate = '';
  let calculatedFees = null;
  let holidays = [];
  let selectedEMIPeriod = 'none'; // Default to no EMI
  let showEMIResults = false;

  onMount(async () => {
    await fetchCategories();
    await fetchCourses();
    await fetchHolidays();
  });

  async function fetchCategories() {
    const querySnapshot = await getDocs(collection(db, 'categories'));
    categories = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    if (categories.length > 0) {
      selectedCategory = categories[0];
      filterCourses();
    }
  }

  async function fetchCourses() {
    const querySnapshot = await getDocs(collection(db, 'courses'));
    courses = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    filterCourses();
  }

  function filterCourses() {
    if (selectedCategory) {
      filteredCourses = courses.filter(course => course.categoryId === selectedCategory.id);
    } else {
      filteredCourses = courses;
    }
    if (filteredCourses.length > 0) {
      selectedCourse = filteredCourses[0];
      studentJoinDate = selectedCourse.courseStartDate;
    } else {
      selectedCourse = null;
      studentJoinDate = '';
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

  function calculateEMI(totalAmount, periods, isWeekly = false) {
    const periodsPerYear = isWeekly ? 52 : 12;
    const emi = totalAmount / periods;
    return emi;
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

    let emiOptions = [];
    if (selectedEMIPeriod !== 'none') {
      if (selectedEMIPeriod === 'monthly') {
        emiOptions = [3, 6, 9, 12].map(months => ({
          period: `${months} months`,
          emi: calculateEMI(feesWithTax, months)
        }));
      } else if (selectedEMIPeriod === 'weekly') {
        emiOptions = [4, 6, 8, 12].map(weeks => ({
          period: `${weeks} weeks`,
          emi: calculateEMI(feesWithTax, weeks, true)
        }));
      }
    }

    calculatedFees = {
      course: selectedCourse.name,
      totalClasses,
      totalHours,
      totalFees,
      feesWithTax,
      courseStartDate: courseStartDate.toDateString(),
      studentJoinDate: joinDate.toDateString(),
      courseEndDate: courseEndDate.toDateString(),
      emiOptions,
    };

    showEMIResults = selectedEMIPeriod !== 'none';
  }
</script>

<main class="container mx-auto p-4 max-w-2xl">
  <h1 class="text-3xl font-bold mb-6 text-center text-blue-600">Course Fee Calculator</h1>

  <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
    <div class="mb-4">
      <label for="category" class="block text-gray-700 text-sm font-bold mb-2">Category:</label>
      <select 
        id="category" 
        bind:value={selectedCategory} 
        on:change={filterCourses} 
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
      >
        <option value={null}>All Categories</option>
        {#each categories as category}
          <option value={category}>{category.name}</option>
        {/each}
      </select>
    </div>

    <div class="mb-4">
      <label for="course" class="block text-gray-700 text-sm font-bold mb-2">Course:</label>
      <select 
        id="course" 
        bind:value={selectedCourse} 
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
      >
        {#each filteredCourses as course}
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

    <div class="mb-4">
      <label for="emiPeriod" class="block text-gray-700 text-sm font-bold mb-2">EMI Options:</label>
      <select id="emiPeriod" bind:value={selectedEMIPeriod} class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
        <option value="none">No EMI</option>
        <option value="monthly">Monthly EMI</option>
        <option value="weekly">Weekly EMI</option>
      </select>
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
      
      {#if showEMIResults}
        <div class="mt-6">
          <h3 class="text-xl font-bold mb-2">EMI Options:</h3>
          <table class="w-full">
            <thead>
              <tr>
                <th class="text-left">Period</th>
                <th class="text-right">EMI Amount</th>
              </tr>
            </thead>
            <tbody>
              {#each calculatedFees.emiOptions as option}
                <tr>
                  <td>{option.period}</td>
                  <td class="text-right">¥{option.emi.toFixed(2)}</td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      {/if}
    </div>
  {/if}
</main>

<style>
  :global(body) {
    background-color: #f3f4f6;
  }
</style>