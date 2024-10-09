<script>
  import { onMount } from 'svelte';
  import { db } from '$lib/firebase';
  import { collection, getDocs } from 'firebase/firestore';
  import { jsPDF } from 'jspdf';
  import 'jspdf-autotable';

  let categories = [];
  let courses = [];
  let holidays = [];
  let selectedCategory = null;
  let selectedCourse = null;
  let studentJoinDate = '';
  let calculatedFees = null;
  let paymentOption = 'full';
  let installmentEndDate = '';
  let activeTab = 'calculator';

  const REGISTRATION_FEE = 5000;
  const TAX_RATE = 0.1; // 10% tax rate

  onMount(async () => {
    await fetchCategories();
    await fetchCourses();
    await fetchHolidays();
  });

  async function fetchCategories() {
    const querySnapshot = await getDocs(collection(db, 'categories'));
    categories = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
  }

  async function fetchCourses() {
    const querySnapshot = await getDocs(collection(db, 'courses'));
    courses = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
  }

  async function fetchHolidays() {
    const querySnapshot = await getDocs(collection(db, 'holidays'));
    holidays = querySnapshot.docs.map(doc => doc.data());
  }

  function isHoliday(date) {
    const holiday = holidays.find(h => h.month === date.getMonth() + 1);
    return holiday ? holiday.days.includes(date.getDate()) : false;
  }

  function isLeapYear(year) {
    return (year % 4 === 0 && year % 100 !== 0) || (year % 400 === 0);
  }

  function calculateFees() {
    if (!selectedCourse || !studentJoinDate) {
      alert('Please select a course and join date');
      return;
    }

    const courseStartDate = new Date(selectedCourse.courseStartDate);
    const joinDate = new Date(studentJoinDate);
    const courseEndDate = new Date(selectedCourse.courseEndDate);
    const endDate = paymentOption === 'installment' && installmentEndDate
      ? new Date(installmentEndDate)
      : courseEndDate;

    if (joinDate < courseStartDate) {
      alert('Join date cannot be earlier than the course start date');
      return;
    }

    if (joinDate > courseEndDate) {
      alert('Join date is after the course has ended');
      return;
    }

    if (paymentOption === 'installment' && new Date(installmentEndDate) > courseEndDate) {
      alert('Installment end date cannot be after the course end date');
      return;
    }

    let currentDate = new Date(Math.max(joinDate, courseStartDate));
    let monthlyBreakdown = [];
    let totalHours = 0;
    let currentMonth = currentDate.getMonth();
    let currentYear = currentDate.getFullYear();
    let currentMonthHours = 0;
    let currentMonthDays = 0;

    while (currentDate <= endDate) {
      if (selectedCourse.classDays.includes(currentDate.getDay()) && !isHoliday(currentDate)) {
        totalHours += selectedCourse.classHours;
        currentMonthHours += selectedCourse.classHours;
        currentMonthDays++;
      }

      // Check if it's the last day of the month or the last day of the course
      const nextDate = new Date(currentDate);
      nextDate.setDate(nextDate.getDate() + 1);
      const isLastDayOfMonth = currentDate.getMonth() !== nextDate.getMonth();
      const isLastDayOfCourse = currentDate.getTime() === endDate.getTime();

      if (isLastDayOfMonth || isLastDayOfCourse) {
        if (currentMonthDays > 0) {
          monthlyBreakdown.push({
            month: new Date(currentYear, currentMonth, 1).toLocaleString('default', { month: 'long', year: 'numeric' }),
            hours: currentMonthHours,
            fees: currentMonthHours * selectedCourse.feePerHour
          });
        }
        currentMonth = nextDate.getMonth();
        currentYear = nextDate.getFullYear();
        currentMonthHours = 0;
        currentMonthDays = 0;
      }

      currentDate.setDate(currentDate.getDate() + 1);
    }

    const courseFees = monthlyBreakdown.reduce((total, month) => total + month.fees, 0);
    const registrationFee = paymentOption === 'full' ? 0 : REGISTRATION_FEE;
    const subTotal = courseFees + registrationFee;
    const taxAmount = subTotal * TAX_RATE;
    const totalFeesWithTax = subTotal + taxAmount;

    calculatedFees = {
      course: selectedCourse.name,
      totalHours,
      courseFees,
      registrationFee,
      subTotal,
      taxAmount,
      totalFeesWithTax,
      courseStartDate: courseStartDate.toDateString(),
      studentJoinDate: joinDate.toDateString(),
      courseEndDate: endDate.toDateString(),
      courseDays: selectedCourse.classDays.map(day => ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'][day]).join(', '),
      courseDuration: selectedCourse.durationMonths,
      courseTime: selectedCourse.courseTime,
      monthlyBreakdown
    };

    if (paymentOption === 'installment') {
      const installmentMonths = monthlyBreakdown.length;
      calculatedFees.installmentAmount = totalFeesWithTax / installmentMonths;
      calculatedFees.installments = installmentMonths;
    }

    activeTab = 'results';
  }
  

  function exportToPDF() {
    const doc = new jsPDF();
    
    // Add title
    doc.setFontSize(18);
    doc.text('Course Fee Calculation', 14, 22);

    // Add course details
    doc.setFontSize(12);
    doc.text(`Course: ${calculatedFees.course}`, 14, 32);
    doc.text(`Total Hours: ${calculatedFees.totalHours}`, 14, 40);
    doc.text(`Course Start: ${calculatedFees.courseStartDate}`, 14, 48);
    doc.text(`Student Join: ${calculatedFees.studentJoinDate}`, 14, 56);
    doc.text(`Course End: ${calculatedFees.courseEndDate}`, 14, 64);
    doc.text(`Course Days: ${calculatedFees.courseDays}`, 14, 72);
    doc.text(`Course Time: ${calculatedFees.courseTime}`, 14, 80);

    // Add fee breakdown
    doc.text('Fee Breakdown:', 14, 92);
    doc.autoTable({
      startY: 96,
      head: [['Item', 'Amount (¥)']],
      body: [
        ['Course Fees', calculatedFees.courseFees.toFixed(2)],
        ['Registration Fee', calculatedFees.registrationFee.toFixed(2)],
        ['Sub Total', calculatedFees.subTotal.toFixed(2)],
        ['Tax (10%)', calculatedFees.taxAmount.toFixed(2)],
        ['Total Fees', calculatedFees.totalFeesWithTax.toFixed(2)]
      ],
    });

    // Add installment details if applicable
    if (paymentOption === 'installment') {
      doc.text('Installment Details:', 14, doc.lastAutoTable.finalY + 10);
      doc.text(`Number of Installments: ${calculatedFees.installments}`, 14, doc.lastAutoTable.finalY + 18);
      doc.text(`Monthly Installment: ¥${calculatedFees.installmentAmount.toFixed(2)}`, 14, doc.lastAutoTable.finalY + 26);
    }

    // Add monthly breakdown
    doc.addPage();
    doc.text('Monthly Breakdown:', 14, 20);
    doc.autoTable({
      startY: 24,
      head: [['Month', 'Hours', 'Fees (¥)']],
      body: calculatedFees.monthlyBreakdown.map(m => [m.month, m.hours, m.fees.toFixed(2)]),
    });

    // Save the PDF
    doc.save('course_fee_calculation.pdf');
  }

  $: filteredCourses = selectedCategory 
    ? courses.filter(course => course.categoryId === selectedCategory.id)
    : [];
</script>

<div class="container mx-auto p-4 max-w-2xl">
  <h1 class="text-3xl font-bold mb-6 text-center text-blue-600">Course Fee Calculator</h1>
  
  <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
    <div class="mb-4">
      <button
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded mr-2"
        class:bg-blue-700={activeTab === 'calculator'}
        on:click={() => activeTab = 'calculator'}
      >
        Calculator
      </button>
      <button
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
        class:bg-blue-700={activeTab === 'results'}
        on:click={() => activeTab = 'results'}
        disabled={!calculatedFees}
      >
        Results
      </button>
    </div>

    {#if activeTab === 'calculator'}
      <h2 class="text-xl font-bold mb-4">Categories</h2>
      <div class="mb-4">
        <label for="category" class="block text-gray-700 text-sm font-bold mb-2">Select Category:</label>
        <select
          id="category"
          bind:value={selectedCategory}
          on:change={() => selectedCourse = null}
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        >
          <option value={null}>Select a category</option>
          {#each categories as category}
            <option value={category}>{category.name}</option>
          {/each}
        </select>
      </div>

      <h2 class="text-xl font-bold mb-4">Courses</h2>
      <div class="mb-4">
        <label for="course" class="block text-gray-700 text-sm font-bold mb-2">Select Course:</label>
        <select
          id="course"
          bind:value={selectedCourse}
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        >
          <option value={null}>Select a course</option>
          {#each filteredCourses as course}
            <option value={course}>{course.name}</option>
          {/each}
        </select>
      </div>

      {#if selectedCourse}
        <div class="mb-4">
          <label for="courseStartDate" class="block text-gray-700 text-sm font-bold mb-2">Course Start Date:</label>
          <input
            type="date"
            id="courseStartDate"
            value={selectedCourse.courseStartDate}
            readonly
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          />
        </div>
        <div class="mb-4">
          <label for="courseEndDate" class="block text-gray-700 text-sm font-bold mb-2">Course End Date:</label>
          <input
            type="date"
            id="courseEndDate"
            value={selectedCourse.courseEndDate}
            readonly
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          />
        </div>
        <div class="mb-4">
          <label for="courseDuration" class="block text-gray-700 text-sm font-bold mb-2">Course Duration:</label>
          <input
            type="text"
            id="courseDuration"
            value={`${selectedCourse.durationMonths} months`}
            readonly
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          />
        </div>
        <div class="mb-4">
          <label for="courseTime" class="block text-gray-700 text-sm font-bold mb-2">Course Time:</label>
          <input
            type="text"
            id="courseTime"
            value={selectedCourse.courseTime}
            readonly
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          />
        </div>
      {/if}

      <h2 class="text-xl font-bold mb-4">Course Details</h2>
      
      <div class="mb-4">
        <label for="studentJoinDate" class="block text-gray-700 text-sm font-bold mb-2">Student Join Date:</label>
        <input
          type="date"
          id="studentJoinDate"
          bind:value={studentJoinDate}
          min={selectedCourse?.courseStartDate}
          max={selectedCourse?.courseEndDate}
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        />
      </div>

      <div class="mb-4">
        <label for="paymentOption" class="block text-gray-700 text-sm font-bold mb-2">Payment Option:</label>
        <select
          id="paymentOption"
          bind:value={paymentOption}
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        >
          <option value="full">Full Payment</option>
          <option value="installment">Pay in Installments</option>
        </select>
      </div>

      {#if paymentOption === 'installment'}
        <div class="mb-4">
          <label for="installmentEndDate" class="block text-gray-700 text-sm font-bold mb-2">Installment End Date:</label>
          <input
            type="date"
            id="installmentEndDate"
            bind:value={installmentEndDate}
            min={studentJoinDate}
            max={selectedCourse?.courseEndDate}
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          />
        </div>
      {/if}

      <div class="flex items-center justify-center">
        <button
          on:click={calculateFees}
          class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
        >
          Calculate Fees
        
        </button>
      </div>
    {:else if activeTab === 'results' && calculatedFees}
      <h2 class="text-2xl font-bold mb-4 text-center text-blue-600">Calculation Results</h2>
      <div class="grid grid-cols-2 gap-4">
        <div class="font-bold">Course:</div>
        <div>{calculatedFees.course}</div>
        <div class="font-bold">Total Hours:</div>
        <div>{calculatedFees.totalHours}</div>
        <div class="font-bold">Course Fees:</div>
        <div>¥{calculatedFees.courseFees.toFixed(2)}</div>
        {#if calculatedFees.registrationFee > 0}
          <div class="font-bold">Registration Fee:</div>
          <div>¥{calculatedFees.registrationFee.toFixed(2)}</div>
        {/if}
        <div class="font-bold">Sub Total:</div>
        <div>¥{calculatedFees.subTotal.toFixed(2)}</div>
        <div class="font-bold">Tax (10%):</div>
        <div>¥{calculatedFees.taxAmount.toFixed(2)}</div>
        <div class="font-bold">Total Fees (with Tax):</div>
        <div>¥{calculatedFees.totalFeesWithTax.toFixed(2)}</div>
        <div class="font-bold">Course Start Date:</div>
        <div>{calculatedFees.courseStartDate}</div>
        <div class="font-bold">Student Join Date:</div>
        <div>{calculatedFees.studentJoinDate}</div>
        <div class="font-bold">Course End Date:</div>
        <div>{calculatedFees.courseEndDate}</div>
        <div class="font-bold">Course Days:</div>
        <div>{calculatedFees.courseDays}</div>
        <div class="font-bold">Course Duration:</div>
        <div>{calculatedFees.courseDuration} months</div>
        <div class="font-bold">Course Time:</div>
        <div>{calculatedFees.courseTime}</div>
      </div>

      {#if paymentOption === 'installment' && calculatedFees.installments}
        <div class="mt-6">
          <h3 class="text-xl font-bold mb-2">Installment Details</h3>
          <div class="grid grid-cols-2 gap-4">
            <div class="font-bold">Installment Duration:</div>
            <div>{calculatedFees.installments} months</div>
            <div class="font-bold">Monthly Installment:</div>
            <div>¥{calculatedFees.installmentAmount.toFixed(2)}</div>
          </div>
        </div>
      {/if}

      <div class="mt-6">
        <h3 class="text-xl font-bold mb-2">Monthly Breakdown</h3>
        <table class="w-full border-collapse">
          <thead>
            <tr>
              <th class="border p-2">Month</th>
              <th class="border p-2">Hours</th>
              <th class="border p-2">Fees</th>
            </tr>
          </thead>
          <tbody>
            {#each calculatedFees.monthlyBreakdown as month}
              <tr>
                <td class="border p-2">{month.month}</td>
                <td class="border p-2">{month.hours}</td>
                <td class="border p-2">¥{month.fees.toFixed(2)}</td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>

      <div class="mt-6 flex justify-center">
        <button
          on:click={exportToPDF}
          class="bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
        >
          Export to PDF
        </button>
      </div>
    {/if}
  </div>
</div>

<style>
  :global(body) {
    background-color: #f3f4f6;
  }
</style>