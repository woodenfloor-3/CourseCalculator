<script>
  import { onMount } from "svelte";
  import { db } from "$lib/firebase";
  import { collection, getDocs } from "firebase/firestore";

  let categories = [];
  let selectedCategory = null;
  let courses = [];
  let filteredCourses = [];
  let selectedCourse = null;
  let studentJoinDate = "";
  let calculatedFees = null;
  let holidays = [];
  let paymentOption = "full";
  let installmentOptions = [];
  let selectedInstallmentOption = "";
  let showInstallmentResults = false;
  let installmentDuration = "";

  const REGISTRATION_FEE = 5000;
  const TAX_RATE = 0.1; // 10% tax rate

  const daysOfWeek = [
    "Sunday",
    "Monday",
    "Tuesday",
    "Wednesday",
    "Thursday",
    "Friday",
    "Saturday",
  ];

  onMount(async () => {
    await fetchCategories();
    await fetchCourses();
    await fetchHolidays();

    const storedCategory = localStorage.getItem("selectedCategory");
    if (storedCategory) {
      selectedCategory = JSON.parse(storedCategory);
      filterCourses();
    }
  });

  async function fetchCategories() {
    const querySnapshot = await getDocs(collection(db, "categories"));
    categories = querySnapshot.docs.map((doc) => ({
      id: doc.id,
      ...doc.data(),
    }));
    if (categories.length > 0 && !selectedCategory) {
      selectedCategory = categories[0];
      filterCourses();
    }
  }

  async function fetchCourses() {
    const querySnapshot = await getDocs(collection(db, "courses"));
    courses = querySnapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
    filterCourses();
  }

  function filterCourses() {
    if (selectedCategory) {
      filteredCourses = courses.filter(
        (course) => course.categoryId === selectedCategory.id
      );
    } else {
      filteredCourses = courses;
    }
    if (filteredCourses.length > 0) {
      selectedCourse = filteredCourses[0];
      studentJoinDate = selectedCourse.courseStartDate;
      updateInstallmentOptions();
    } else {
      selectedCourse = null;
      studentJoinDate = "";
      installmentOptions = [];
    }
    paymentOption = "full";
    selectedInstallmentOption = "";
    showInstallmentResults = false;
    localStorage.setItem("selectedCategory", JSON.stringify(selectedCategory));
  }

  async function fetchHolidays() {
    const querySnapshot = await getDocs(collection(db, "holidays"));
    holidays = querySnapshot.docs.map((doc) => doc.data());
  }

  function isHoliday(date) {
    const holiday = holidays.find((h) => h.month === date.getMonth() + 1);
    return holiday ? holiday.days.includes(date.getDate()) : false;
  }

  function updateInstallmentOptions() {
    if (selectedCourse) {
      const courseDurationMonths = selectedCourse.durationMonths;
      installmentOptions = Array.from(
        { length: courseDurationMonths },
        (_, i) => (i + 1).toString()
      );
    } else {
      installmentOptions = [];
    }
  }

  function calculateFees() {
    if (!selectedCourse || !studentJoinDate) {
      alert("Please select a course and join date");
      return;
    }

    const courseStartDate = new Date(selectedCourse.courseStartDate);
    const joinDate = new Date(studentJoinDate);
    const courseEndDate = new Date(selectedCourse.courseEndDate);

    if (joinDate < courseStartDate) {
      alert("Join date cannot be earlier than the course start date");
      return;
    }

    if (joinDate > courseEndDate) {
      alert("Join date is after the course has ended");
      return;
    }

    let currentDate = new Date(joinDate);
    let totalClasses = 0;
    let totalHours = 0;

    while (currentDate <= courseEndDate) {
      if (
        selectedCourse.classDays.includes(currentDate.getDay()) &&
        !isHoliday(currentDate)
      ) {
        totalClasses++;
        totalHours += selectedCourse.classHours;
      }
      currentDate.setDate(currentDate.getDate() + 1);
    }

    const courseFees = totalHours * selectedCourse.feePerHour;
    const registrationFee =
      paymentOption === "installment" ? REGISTRATION_FEE : 0;
    const totalFeesWithRegistration = courseFees + registrationFee;
    const taxAmount = totalFeesWithRegistration * TAX_RATE;
    const totalFeesWithTax = totalFeesWithRegistration + taxAmount;

    calculatedFees = {
      course: selectedCourse.name,
      totalClasses,
      totalHours,
      courseFees,
      registrationFee,
      taxAmount,
      totalFeesWithTax,
      courseStartDate: courseStartDate.toDateString(),
      studentJoinDate: joinDate.toDateString(),
      courseEndDate: courseEndDate.toDateString(),
      courseDays: selectedCourse.classDays
        .map((day) => daysOfWeek[day])
        .join(", "),
      courseDuration: selectedCourse.durationMonths,
    };

    if (paymentOption === "installment") {
      calculatedFees.totalFeesWithRegistration = totalFeesWithRegistration;
    }

    if (paymentOption === "installment" && selectedInstallmentOption) {
      const installments = parseInt(selectedInstallmentOption);
      calculatedFees.installmentAmount = calculateInstallment(
        totalFeesWithTax,
        installments
      );
      calculatedFees.installments = installments;
    }

    showInstallmentResults =
      paymentOption === "installment" && selectedInstallmentOption !== "";
  }

  function calculateInstallment(totalAmount, installments) {
    return totalAmount / installments;
  }

  $: if (selectedCourse) {
    updateInstallmentOptions();
  }

  $: if (selectedCourse && studentJoinDate && paymentOption) {
    calculateFees();
  }

  $: if (paymentOption === "full") {
    selectedInstallmentOption = "";
    showInstallmentResults = false;
  }

  let isFormValid = false;

  $: isFormValid =
    selectedCourse &&
    studentJoinDate &&
    (paymentOption === "full" ||
      (paymentOption === "installment" && selectedInstallmentOption));
</script>

<main class="container mx-auto p-4 max-w-2xl">
  <h1 class="text-3xl font-bold mb-6 text-center text-blue-600">
    Course Fee Calculator
  </h1>

  <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
    <div class="mb-4">
      <label for="category" class="block text-gray-700 text-sm font-bold mb-2"
        >Category:</label
      >
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
      <label for="course" class="block text-gray-700 text-sm font-bold mb-2"
        >Course:</label
      >
      <select
        id="course"
        bind:value={selectedCourse}
        on:change={updateInstallmentOptions}
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
      >
        {#each filteredCourses as course}
          <option value={course}>{course.name}</option>
        {/each}
      </select>
    </div>

    <div class="mb-4">
      <label
        for="courseEndDate"
        class="block text-gray-700 text-sm font-bold mb-2"
        >Course End Date:</label
      >
      <input
        type="date"
        id="courseEndDate"
        value={selectedCourse?.courseEndDate}
        readonly
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
      />
    </div>

    <div class="mb-4">
      <label
        for="studentJoinDate"
        class="block text-gray-700 text-sm font-bold mb-2"
        >Student Join Date:</label
      >
      <input
        type="date"
        id="studentJoinDate"
        bind:value={studentJoinDate}
        min={selectedCourse?.courseStartDate}
        max={selectedCourse?.courseEndDate}
        placeholder="YYYY-MM-DD"
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
      />
    </div>

    <div class="mb-4">
      <label
        for="paymentOption"
        class="block text-gray-700 text-sm font-bold mb-2"
        >Payment Option:</label
      >
      <select
        id="paymentOption"
        bind:value={paymentOption}
        class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
      >
        <option value="full">Full Payment</option>
        <option value="installment">Pay in Installments</option>
      </select>
    </div>

    {#if paymentOption === "installment"}
      <div class="mb-4">
        <label
          for="installmentOption"
          class="block text-gray-700 text-sm font-bold mb-2"
          >Installment Duration:</label
        >
        <select
          id="installmentOption"
          bind:value={selectedInstallmentOption}
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        >
          <option value="">Select Installment Duration</option>
          {#each installmentOptions as option}
            <option value={option}
              >{option} month{option !== "1" ? "s" : ""}</option
            >
          {/each}
        </select>
      </div>
      <div class="mb-4">
        <label
          for="installmentDuration"
          class="block text-gray-700 text-sm font-bold mb-2"
          >Installment End Date:</label
        >
        <input
          type="date"
          id="installmentDuration"
          bind:value={installmentDuration}
          min={studentJoinDate}
          max={selectedCourse?.courseEndDate}
          placeholder="YYYY-MM-DD"
          class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
        />
      </div>
    {/if}

    <div class="flex items-center justify-center">
      <button
        on:click={calculateFees}
        disabled={!isFormValid}
        class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline
               disabled:bg-gray-400 disabled:cursor-not-allowed"
      >
        Calculate Fees
      </button>
    </div>
  </div>

  {#if calculatedFees}
    <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
      <h2 class="text-2xl font-bold mb-4 text-center text-blue-600">
        Calculation Results
      </h2>
      <div class="grid grid-cols-2 gap-4">
        <div class="font-bold">Course:</div>
        <div>{calculatedFees.course}</div>
        <div class="font-bold">Total Classes:</div>
        <div>{calculatedFees.totalClasses}</div>
        <div class="font-bold">Total Hours:</div>
        <div>{calculatedFees.totalHours}</div>
        <div class="font-bold">Course Fees:</div>
        <div>¥{calculatedFees.courseFees.toFixed(2)}</div>
        {#if paymentOption === "installment"}
          <div class="font-bold">Registration Fee:</div>
          <div>¥{calculatedFees.registrationFee.toFixed(2)}</div>
          <div class="font-bold">Total Fees (with Registration):</div>
          <div>¥{calculatedFees.totalFeesWithRegistration.toFixed(2)}</div>
        {/if}
        <div class="font-bold">Tax Amount (10%):</div>
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
      </div>

      {#if showInstallmentResults}
        <div class="mt-6">
          <h3 class="text-xl font-bold mb-2">Installment Details</h3>
          <div class="grid grid-cols-2 gap-4">
            <div class="font-bold">Installment Duration:</div>
            <div>
              {calculatedFees.installments} month{calculatedFees.installments !==
              1
                ? "s"
                : ""}
            </div>
            <div class="font-bold">Installment Amount:</div>
            <div>¥{calculatedFees.installmentAmount.toFixed(2)}</div>
            <div class="font-bold">Number of Installments:</div>
            <div>{calculatedFees.installments}</div>
          </div>
        </div>
      {/if}
    </div>
  {/if}
</main>
