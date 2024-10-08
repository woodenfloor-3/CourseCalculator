<script>
  import { onMount } from "svelte";
  import { db, auth } from "$lib/firebase";
  import {
    collection,
    addDoc,
    getDocs,
    updateDoc,
    deleteDoc,
    doc,
  } from "firebase/firestore";
  import { onAuthStateChanged } from "firebase/auth";
  import { goto } from "$app/navigation";
  import {
    Trash2,
    Edit,
    Plus,
    Calendar,
    Clock,
    Book,
    JapaneseYen,
    User,
  } from "lucide-svelte";

  let categories = [];
  let courses = [];
  let holidays = [];
  let newCategory = { name: "" };
  let newCourse = {
    name: "",
    categoryId: "",
    classDays: [],
    classHours: 1.5,
    durationMonths: 0,
    feePerHour: 600,
    courseStartDate: "",
    courseEndDate: "",
    timeSlotStart: "07:00",
    timeSlotStartPeriod: "AM",
  };
  let newHoliday = { month: 1, days: "" };
  let user = null;
  let activeTab = "categories";
  let showUpdateModal = false;
  let courseToUpdate = null;

  const daysOfWeek = [
    { id: 0, name: "Sunday" },
    { id: 1, name: "Monday" },
    { id: 2, name: "Tuesday" },
    { id: 3, name: "Wednesday" },
    { id: 4, name: "Thursday" },
    { id: 5, name: "Friday" },
    { id: 6, name: "Saturday" },
  ];

  onMount(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      if (currentUser) {
        user = currentUser;
        fetchCategories();
        fetchCourses();
        fetchHolidays();
      } else {
        goto("/login");
      }
    });

    return unsubscribe;
  });

  async function fetchCategories() {
    const querySnapshot = await getDocs(collection(db, "categories"));
    categories = querySnapshot.docs.map((doc) => ({
      id: doc.id,
      ...doc.data(),
    }));
  }

  async function fetchCourses() {
    const querySnapshot = await getDocs(collection(db, "courses"));
    courses = querySnapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
  }

  async function fetchHolidays() {
    const querySnapshot = await getDocs(collection(db, "holidays"));
    holidays = querySnapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
  }

  async function addCategory() {
    await addDoc(collection(db, "categories"), newCategory);
    fetchCategories();
    newCategory = { name: "" };
  }

  async function deleteCategory(id) {
    const coursesInCategory = courses.filter(
      (course) => course.categoryId === id
    );
    if (coursesInCategory.length > 0) {
      alert(
        "Cannot delete category. There are courses associated with this category."
      );
      return;
    }

    await deleteDoc(doc(db, "categories", id));
    await fetchCategories();
  }

  function calculateTimeSlotEnd(start, hours, period) {
    let [startHour, startMinute] = start.split(':').map(Number);
    if (period === 'PM' && startHour !== 12) startHour += 12;
    if (period === 'AM' && startHour === 12) startHour = 0;

    let totalMinutes = startHour * 60 + startMinute + hours * 60;
    let endHour = Math.floor(totalMinutes / 60) % 24;
    let endMinute = totalMinutes % 60;

    let endPeriod = endHour >= 12 ? 'PM' : 'AM';
    endHour = endHour > 12 ? endHour - 12 : (endHour === 0 ? 12 : endHour);

    return `${endHour.toString().padStart(2, '0')}:${endMinute.toString().padStart(2, '0')} ${endPeriod}`;
  }

  $: newCourse.timeSlot = `${newCourse.timeSlotStart} ${newCourse.timeSlotStartPeriod} - ${calculateTimeSlotEnd(newCourse.timeSlotStart, newCourse.classHours, newCourse.timeSlotStartPeriod)}`;

  function updateCourseToUpdateTimeSlot() {
    if (courseToUpdate) {
      let [start, end] = courseToUpdate.timeSlot.split(' - ');
      let [startTime, startPeriod] = start.split(' ');
      courseToUpdate.timeSlotStart = startTime;
      courseToUpdate.timeSlotStartPeriod = startPeriod;
    }
  }

  $: if (courseToUpdate) {
    courseToUpdate.timeSlot = `${courseToUpdate.timeSlotStart} ${courseToUpdate.timeSlotStartPeriod} - ${calculateTimeSlotEnd(courseToUpdate.timeSlotStart, courseToUpdate.classHours, courseToUpdate.timeSlotStartPeriod)}`;
  }

  async function addCourse() {
    const courseData = {
      ...newCourse,
      createdAt: new Date().toISOString(),
      createdBy: user.email,
    };
    await addDoc(collection(db, "courses"), courseData);
    await fetchCourses();
    resetNewCourse();
  }

  function openUpdateModal(course) {
    courseToUpdate = { ...course };
    updateCourseToUpdateTimeSlot();
    showUpdateModal = true;
  }

  function closeUpdateModal() {
    showUpdateModal = false;
    courseToUpdate = null;
  }

  async function updateCourse() {
    if (!courseToUpdate) return;

    const courseRef = doc(db, "courses", courseToUpdate.id);
    await updateDoc(courseRef, {
      name: courseToUpdate.name,
      categoryId: courseToUpdate.categoryId,
      classDays: courseToUpdate.classDays,
      classHours: courseToUpdate.classHours,
      durationMonths: courseToUpdate.durationMonths,
      feePerHour: courseToUpdate.feePerHour,
      courseStartDate: courseToUpdate.courseStartDate,
      courseEndDate: courseToUpdate.courseEndDate,
      timeSlot: courseToUpdate.timeSlot,
      timeSlotStart: courseToUpdate.timeSlotStart,
      timeSlotStartPeriod: courseToUpdate.timeSlotStartPeriod,
    });
    await fetchCourses();
    closeUpdateModal();
  }

  async function deleteCourse(id) {
    await deleteDoc(doc(db, "courses", id));
    await fetchCourses();
  }

  function resetNewCourse() {
    newCourse = {
      name: "",
      categoryId: "",
      classDays: [],
      classHours: 1.5,
      durationMonths: 0,
      feePerHour: 600,
      courseStartDate: "",
      courseEndDate: "",
      timeSlotStart: "07:00",
      timeSlotStartPeriod: "AM",
    };
  }

  async function addHoliday() {
    const holidayToAdd = {
      month: parseInt(newHoliday.month),
      days: newHoliday.days
        .split(",")
        .map((day) => parseInt(day.trim()))
        .filter((day) => !isNaN(day) && day > 0 && day <= 31),
    };
    await addDoc(collection(db, "holidays"), holidayToAdd);
    newHoliday = { month: 1, days: "" };
    await fetchHolidays();
  }

  async function deleteHoliday(id) {
    await deleteDoc(doc(db, "holidays", id));
    await fetchHolidays();
  }

  const months = [
    "January",
    "February",
    "March",
    "April",
    "May",
    "June",
    "July",
    "August",
    "September",
    "October",
    "November",
    "December",
  ];

  function toggleDay(day) {
    const index = newCourse.classDays.indexOf(day);
    if (index === -1) {
      newCourse.classDays = [...newCourse.classDays, day];
    } else {
      newCourse.classDays = newCourse.classDays.filter((d) => d !== day);
    }
  }

  function toggleUpdateDay(day) {
    if (!courseToUpdate) return;
    const index = courseToUpdate.classDays.indexOf(day);
    if (index === -1) {
      courseToUpdate.classDays = [...courseToUpdate.classDays, day];
    } else {
      courseToUpdate.classDays = courseToUpdate.classDays.filter(
        (d) => d !== day
      );
    }
  }

  function formatDays(days) {
    return days.map((day) => daysOfWeek[day].name).join(", ");
  }

  function calculateDurationMonths() {
    if (newCourse.courseStartDate && newCourse.courseEndDate) {
      const start = new Date(newCourse.courseStartDate);
      const end = new Date(newCourse.courseEndDate);
      const diffTime = Math.abs(end - start);
      const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
      newCourse.durationMonths = Math.round(diffDays / 30.44); // Using average days in a month
    }
  }

  function formatDate(dateString) {
    const date = new Date(dateString);
    return `${date.getDate().toString().padStart(2, "0")}/${(date.getMonth() + 1).toString().padStart(2, "0")}/${date.getFullYear()}`;
  }
</script>

{#if user}
  <main class="container mx-auto p-4 max-w-6xl">
    <h1 class="text-4xl font-bold mb-8 text-center text-blue-600">
      Admin Dashboard
    </h1>

    <div class="bg-white shadow-lg rounded-lg overflow-hidden mb-8">
      <div class="flex border-b">
        <button
          class="flex-1 py-4 px-6 text-center font-semibold {activeTab ===
          'categories'
            ? 'bg-blue-500 text-white'
            : 'hover:bg-gray-100'}"
          on:click={() => (activeTab = "categories")}
        >
          Categories
        </button>
        <button
          class="flex-1 py-4 px-6 text-center font-semibold {activeTab ===
          'courses'
            ? 'bg-blue-500 text-white'
            : 'hover:bg-gray-100'}"
          on:click={() => (activeTab = "courses")}
        >
          Courses
        </button>
        <button
          class="flex-1 py-4 px-6 text-center font-semibold {activeTab ===
          'holidays'
            ? 'bg-blue-500 text-white'
            : 'hover:bg-gray-100'}"
          on:click={() => (activeTab = "holidays")}
        >
          Holidays
        </button>
      </div>

      {#if activeTab === "categories"}
        <div class="p-6">
          <h2 class="text-2xl font-bold mb-4">Manage Categories</h2>
          <form on:submit|preventDefault={addCategory} class="mb-6">
            <div class="flex items-center space-x-4">
              <div class="flex-1">
                <label
                  for="categoryName"
                  class="block text-gray-700 text-sm font-bold mb-2"
                  >Category Name:</label
                >
                <input
                  type="text"
                  id="categoryName"
                  bind:value={newCategory.name}
                  required
                  class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                />
              </div>
              <div class="flex-none">
                <button
                  type="submit"
                  class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline flex items-center"
                >
                  <Plus class="mr-2" size={20} />
                  Add Category
                </button>
              </div>
            </div>
          </form>

          <h3 class="text-xl font-bold mb-4">Existing Categories:</h3>
          <div class="grid gap-4">
            {#each categories as category (category.id)}
              <div
                class="bg-gray-100 p-4 rounded-lg flex items-center justify-between"
              >
                <h4 class="text-lg font-semibold">{category.name}</h4>
                <button
                  on:click={() => deleteCategory(category.id)}
                  class="text-red-500 hover:text-red-700"
                >
                  <Trash2 size={20} />
                </button>
              </div>
            {/each}
          </div>
        </div>
      {/if}

      {#if activeTab === "courses"}
        <div class="p-6">
          <h2 class="text-2xl font-bold mb-4">Manage Courses</h2>
          <form on:submit|preventDefault={addCourse} class="mb-6">
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label
                  for="courseCategory"
                  class="block text-gray-700 text-sm font-bold mb-2"
                  >Category:</label
                >
                <select
                  id="courseCategory"
                  bind:value={newCourse.categoryId}
                  required
                  class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                >
                  <option value="">Select a category</option>
                  {#each categories as category}
                    <option value={category.id}>{category.name}</option>
                  {/each}
                </select>
              </div>
              <div>
                <label
                  for="courseName"
                  class="block text-gray-700 text-sm font-bold mb-2"
                  >Course Name:</label
                >
                <input
                  type="text"
                  id="courseName"
                  bind:value={newCourse.name}
                  required
                  class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                  />
                </div>
                <div class="col-span-2">
                  <fieldset>
                    <legend class="block text-gray-700 text-sm font-bold mb-2"
                      >Class Days:</legend
                    >
                    <div class="flex flex-wrap gap-2">
                      {#each daysOfWeek as day}
                        <label class="inline-flex items-center">
                          <input
                            type="checkbox"
                            class="form-checkbox"
                            checked={newCourse.classDays.includes(day.id)}
                            on:change={() => toggleDay(day.id)}
                          />
                          <span class="ml-2">{day.name}</span>
                        </label>
                      {/each}
                    </div>
                  </fieldset>
                </div>
                <div>
                  <label
                    for="courseClassHours"
                    class="block text-gray-700 text-sm font-bold mb-2"
                    >Class Hours:</label
                  >
                  <input
                    type="number"
                    id="courseClassHours"
                    bind:value={newCourse.classHours}
                    step="0.5"
                    required
                    class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                  />
                </div>
                <div>
                  <label
                    for="courseFeePerHour"
                    class="block text-gray-700 text-sm font-bold mb-2"
                    >Fee Per Hour:</label
                  >
                  <input
                    type="number"
                    id="courseFeePerHour"
                    bind:value={newCourse.feePerHour}
                    required
                    class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                  />
                </div>
                <div>
                  <label
                    for="courseStartDate"
                    class="block text-gray-700 text-sm font-bold mb-2"
                    >Start Date:</label
                  >
                  <input
                    type="date"
                    id="courseStartDate"
                    bind:value={newCourse.courseStartDate}
                    on:change={calculateDurationMonths}
                    required
                    class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                  />
                </div>
                <div>
                  <label
                    for="courseEndDate"
                    class="block text-gray-700 text-sm font-bold mb-2"
                    >End Date:</label
                  >
                  <input
                    type="date"
                    id="courseEndDate"
                    bind:value={newCourse.courseEndDate}
                    on:change={calculateDurationMonths}
                    required
                    class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                  />
                </div>
                <div>
                  <label
                    for="courseDurationMonths"
                    class="block text-gray-700 text-sm font-bold mb-2"
                    >Duration (Months):</label
                  >
                  <input
                    type="number"
                    id="courseDurationMonths"
                    bind:value={newCourse.durationMonths}
                    readonly
                    class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline bg-gray-100"
                  />
                </div>
                <div>
                  <label
                    for="courseTimeSlotStart"
                    class="block text-gray-700 text-sm font-bold mb-2"
                  >Time Slot Start:</label>
                  <div class="flex space-x-2">
                    <input
                      type="text"
                      id="courseTimeSlotStart"
                      bind:value={newCourse.timeSlotStart}
                      placeholder="HH:MM"
                      pattern="([01]?[0-9]|2[0-3]):[0-5][0-9]"
                      required
                      class="shadow appearance-none border rounded py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline flex-grow"
                    />
                    <select
                      bind:value={newCourse.timeSlotStartPeriod}
                      class="shadow appearance-none border rounded py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                    >
                      <option value="AM">AM</option>
                      <option value="PM">PM</option>
                    </select>
                  </div>
                </div>
              </div>
              <div class="mt-6">
                <button
                  type="submit"
                  class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline flex items-center"
                >
                  <Plus class="mr-2" size={20} />
                  Add Course
                </button>
              </div>
            </form>
  
            <h3 class="text-xl font-bold mb-4">Existing Courses:</h3>
            <div class="grid gap-6">
              {#each courses as course (course.id)}
                <div class="bg-gray-100 p-6 rounded-lg">
                  <h4 class="text-xl font-bold mb-2">{course.name}</h4>
                  <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <p class="flex items-center">
                      <Book class="mr-2" size={16} /> Category: {categories.find(
                        (c) => c.id === course.categoryId
                      )?.name || "Uncategorized"}
                    </p>
                    <p class="flex items-center">
                      <Calendar class="mr-2" size={16} /> Class Days: {formatDays(
                        course.classDays
                      )}
                    </p>
                    <p class="flex items-center">
                      <Clock class="mr-2" size={16} /> Class Hours: {course.classHours}
                    </p>
                    <p class="flex items-center">
                      <Calendar class="mr-2" size={16} /> Duration: {course.durationMonths}
                      months
                    </p>
                    <p class="flex items-center">
                      <JapaneseYen class="mr-2" size={16} /> Fee Per Hour: ¥ {course.feePerHour}
                    </p>
                    <p class="flex items-center">
                      <Calendar class="mr-2" size={16} /> Start Date: {formatDate(
                        course.courseStartDate
                      )}
                    </p>
                    <p class="flex items-center">
                      <Calendar class="mr-2" size={16} /> End Date: {formatDate(
                        course.courseEndDate
                      )}
                    </p>
                    <p class="flex items-center">
                      <Clock class="mr-2" size={16} /> Time Slot: {course.timeSlot}
                    </p>
                    <p class="flex items-center">
                      <Calendar class="mr-2" size={16} /> Created: {formatDate(
                        course.createdAt
                      )}
                    </p>
                    <p class="flex items-center">
                      <User class="mr-2" size={16} /> Created by: {course.createdBy}
                    </p>
                  </div>
                  <div class="mt-4 flex justify-end space-x-2">
                    <button
                      on:click={() => openUpdateModal(course)}
                      class="bg-yellow-500 hover:bg-yellow-600 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline flex items-center"
                    >
                      <Edit class="mr-2" size={16} />
                      Update
                    </button>
                    <button
                      on:click={() => deleteCourse(course.id)}
                      class="bg-red-500 hover:bg-red-600 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline flex items-center"
                    >
                      <Trash2 class="mr-2" size={16} />
                      Delete
                    </button>
                  </div>
                </div>
              {/each}
            </div>
          </div>
        {/if}
  
        {#if activeTab === "holidays"}
          <div class="p-6">
            <h2 class="text-2xl font-bold mb-4">Manage Holidays</h2>
            <form on:submit|preventDefault={addHoliday} class="mb-6">
              <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                <div>
                  <label
                    for="holidayMonth"
                    class="block text-gray-700 text-sm font-bold mb-2"
                    >Month:</label
                  >
                  <select
                    id="holidayMonth"
                    bind:value={newHoliday.month}
                    class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                  >
                    {#each months as month, index}
                      <option value={index + 1}>{month}</option>
                    {/each}
                  </select>
                </div>
                <div>
                  <label
                    for="holidayDays"
                    class="block text-gray-700 text-sm font-bold mb-2"
                    >Days (comma-separated):</label
                  >
                  <input
                    type="text"
                    id="holidayDays"
                    bind:value={newHoliday.days}
                    required
                    placeholder="1, 2, 3"
                    class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                  />
                </div>
              </div>
              <div class="mt-6">
                <button
                  type="submit"
                  class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline flex items-center"
                >
                  <Plus class="mr-2" size={20} />
                  Add Holiday
                </button>
              </div>
            </form>
  
            <h3 class="text-xl font-bold mb-4">Existing Holidays:</h3>
            <div class="grid gap-4">
              {#each holidays as holiday (holiday.id)}
                <div
                  class="bg-gray-100 p-4 rounded-lg flex items-center justify-between"
                >
                  <p class="font-semibold">
                    {months[holiday.month - 1]}: {holiday.days.join(", ")}
                  </p>
                  <button
                    on:click={() => deleteHoliday(holiday.id)}
                    class="text-red-500 hover:text-red-700"
                  >
                    <Trash2 size={20} />
                  </button>
                </div>
              {/each}
            </div>
          </div>
        {/if}
      </div>
    </main>
  {/if}
  {#if showUpdateModal && courseToUpdate}
  <div
    class="fixed inset-0 bg-gray-600 bg-opacity-50 overflow-y-auto h-full w-full flex items-center justify-center"
    id="update-course-modal"
  >
    <div class="bg-white rounded-lg shadow-xl p-6 w-full max-w-md">
      <h3 class="text-2xl font-bold mb-4 text-center text-gray-800">
        Update Course
      </h3>
      <form on:submit|preventDefault={updateCourse} class="space-y-4">
        <div>
          <label for="updateCourseName" class="block text-gray-700 font-bold mb-2">
            Course Name:
          </label>
          <input
            type="text"
            id="updateCourseName"
            bind:value={courseToUpdate.name}
            required
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
        </div>

        <div>
          <label for="updateCourseCategory" class="block text-gray-700 font-bold mb-2">
            Category:
          </label>
          <select
            id="updateCourseCategory"
            bind:value={courseToUpdate.categoryId}
            required
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            {#each categories as category}
              <option value={category.id}>{category.name}</option>
            {/each}
          </select>
        </div>

        <div>
          <fieldset>
            <legend class="block text-gray-700 font-bold mb-2">Class Days:</legend>
            <div class="flex flex-wrap gap-2">
              {#each daysOfWeek as day}
                <label class="inline-flex items-center">
                  <input
                    type="checkbox"
                    class="form-checkbox"
                    checked={courseToUpdate.classDays.includes(day.id)}
                    on:change={() => toggleUpdateDay(day.id)}
                  />
                  <span class="ml-2">{day.name}</span>
                </label>
              {/each}
            </div>
          </fieldset>
        </div>

        <div>
          <label for="updateCourseClassHours" class="block text-gray-700 font-bold mb-2">
            Class Hours:
          </label>
          <input
            type="number"
            id="updateCourseClassHours"
            bind:value={courseToUpdate.classHours}
            step="0.5"
            required
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
        </div>

        <div>
          <label for="updateCourseFeePerHour" class="block text-gray-700 font-bold mb-2">
            Fee Per Hour:
          </label>
          <input
            type="number"
            id="updateCourseFeePerHour"
            bind:value={courseToUpdate.feePerHour}
            required
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
        </div>

        <div>
          <label for="updateCourseStartDate" class="block text-gray-700 font-bold mb-2">
            Start Date:
          </label>
          <input
            type="date"
            id="updateCourseStartDate"
            bind:value={courseToUpdate.courseStartDate}
            required
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
        </div>

        <div>
          <label for="updateCourseEndDate" class="block text-gray-700 font-bold mb-2">
            End Date:
          </label>
          <input
            type="date"
            id="updateCourseEndDate"
            bind:value={courseToUpdate.courseEndDate}
            required
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
        </div>

        <div>
          <label for="updateCourseTimeSlotStart" class="block text-gray-700 font-bold mb-2">
            Time Slot Start:
          </label>
          <div class="flex space-x-2">
            <input
              type="time"
              id="updateCourseTimeSlotStart"
              bind:value={courseToUpdate.timeSlotStart}
              required
              class="flex-grow px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            />
            <select
              bind:value={courseToUpdate.timeSlotStartPeriod}
              class="px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            >
              <option value="AM">AM</option>
              <option value="PM">PM</option>
            </select>
          </div>
        </div>

        <div>
          <label for="updateCourseTimeSlot" class="block text-gray-700 font-bold mb-2">
            Time Slot:
          </label>
          <input
            type="text"
            id="updateCourseTimeSlot"
            value={courseToUpdate.timeSlot}
            readonly
            class="w-full px-3 py-2 border border-gray-300 rounded-md bg-gray-100"
          />
        </div>

        <div class="flex justify-end space-x-2 mt-6">
          <button
            type="button"
            on:click={closeUpdateModal}
            class="px-4 py-2 bg-gray-500 text-white rounded-md hover:bg-gray-600 focus:outline-none focus:ring-2 focus:ring-gray-500"
          >
            Cancel
          </button>
          <button
            type="submit"
            class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            Update
          </button>
        </div>
      </form>
    </div>
  </div>
{/if}

<style>
  :global(body) {
    background-color: #f3f4f6;
  }
</style>