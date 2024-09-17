<script>
    import { onMount } from 'svelte';
    import { db, auth } from '$lib/firebase';
    import { collection, addDoc, getDocs, updateDoc, deleteDoc, doc } from 'firebase/firestore';
    import { onAuthStateChanged } from 'firebase/auth';
    import { goto } from '$app/navigation';
  
    let courses = [];
    let holidays = [];
    let newCourse = {
      name: '',
      weeklyClasses: 3,
      classDays: [1, 2, 4],
      classHours: 1.5,
      durationMonths: 11,
      feePerHour: 600,
      courseStartDate: '',
      timeSlot: ''
    };
    let newHoliday = { month: 1, days: '' };
    let user = null;
  
    onMount(() => {
      const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
        if (currentUser) {
          user = currentUser;
          fetchCourses();
          fetchHolidays();
        } else {
          goto('/login');
        }
      });
  
      return unsubscribe;
    });
  
    async function fetchCourses() {
      const querySnapshot = await getDocs(collection(db, 'courses'));
      courses = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    }
  
    async function fetchHolidays() {
      const querySnapshot = await getDocs(collection(db, 'holidays'));
      holidays = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    }
  
    async function addCourse() {
      await addDoc(collection(db, 'courses'), newCourse);
      fetchCourses();
      resetNewCourse();
    }
  
    async function updateCourse(id) {
      const courseRef = doc(db, 'courses', id);
      await updateDoc(courseRef, courses.find(c => c.id === id));
      fetchCourses();
    }
  
    async function deleteCourse(id) {
      await deleteDoc(doc(db, 'courses', id));
      fetchCourses();
    }
  
    function resetNewCourse() {
      newCourse = {
        name: '',
        weeklyClasses: 3,
        classDays: [1, 2, 4],
        classHours: 1.5,
        durationMonths: 11,
        feePerHour: 600,
        courseStartDate: '',
        timeSlot: ''
      };
    }
  
    async function addHoliday() {
      const holidayToAdd = {
        month: parseInt(newHoliday.month),
        days: newHoliday.days.split(',').map(day => parseInt(day.trim())).filter(day => !isNaN(day) && day > 0 && day <= 31)
      };
      await addDoc(collection(db, 'holidays'), holidayToAdd);
      newHoliday = { month: 1, days: '' };
      fetchHolidays();
    }
  
    async function deleteHoliday(id) {
      await deleteDoc(doc(db, 'holidays', id));
      fetchHolidays();
    }
  
    const months = [
      'January', 'February', 'March', 'April', 'May', 'June',
      'July', 'August', 'September', 'October', 'November', 'December'
    ];
  </script>
  
  {#if user}
    <main class="container mx-auto p-4 max-w-4xl">
      <h1 class="text-3xl font-bold mb-6 text-center text-blue-600">Admin Dashboard</h1>
  
      <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
        <h2 class="text-2xl font-bold mb-4">Manage Courses</h2>
        <form on:submit|preventDefault={addCourse} class="mb-4">
          <div class="grid grid-cols-2 gap-4">
            <div>
              <label for="name" class="block text-gray-700 text-sm font-bold mb-2">Course Name:</label>
              <input type="text" id="name" bind:value={newCourse.name} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
            <div>
              <label for="weeklyClasses" class="block text-gray-700 text-sm font-bold mb-2">Weekly Classes:</label>
              <input type="number" id="weeklyClasses" bind:value={newCourse.weeklyClasses} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
            <div>
              <label for="classDays" class="block text-gray-700 text-sm font-bold mb-2">Class Days (comma-separated):</label>
              <input type="text" id="classDays" bind:value={newCourse.classDays} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
            <div>
              <label for="classHours" class="block text-gray-700 text-sm font-bold mb-2">Class Hours:</label>
              <input type="number" id="classHours" bind:value={newCourse.classHours} step="0.5" required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
            <div>
              <label for="durationMonths" class="block text-gray-700 text-sm font-bold mb-2">Duration (Months):</label>
              <input type="number" id="durationMonths" bind:value={newCourse.durationMonths} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
            <div>
              <label for="feePerHour" class="block text-gray-700 text-sm font-bold mb-2">Fee Per Hour:</label>
              <input type="number" id="feePerHour" bind:value={newCourse.feePerHour} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
            <div>
              <label for="courseStartDate" class="block text-gray-700 text-sm font-bold mb-2">Start Date:</label>
              <input type="date" id="courseStartDate" bind:value={newCourse.courseStartDate} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
            <div>
              <label for="timeSlot" class="block text-gray-700 text-sm font-bold mb-2">Time Slot:</label>
              <input type="text" id="timeSlot" bind:value={newCourse.timeSlot} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
          </div>
          <div class="mt-4">
            <button type="submit" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline">
              Add Course
            </button>
          </div>
        </form>
  
        <h3 class="text-xl font-bold mb-2">Existing Courses:</h3>
        {#each courses as course (course.id)}
          <div class="mb-4 p-4 border rounded">
            <h4 class="text-lg font-bold">{course.name}</h4>
            <p>Weekly Classes: {course.weeklyClasses}</p>
            <p>Class Days: {course.classDays.join(', ')}</p>
            <p>Class Hours: {course.classHours}</p>
            <p>Duration: {course.durationMonths} months</p>
            <p>Fee Per Hour: ¥{course.feePerHour}</p>
            <p>Start Date: {course.courseStartDate}</p>
            <p>Time Slot: {course.timeSlot}</p>
            <div class="mt-2">
              <button on:click={() => updateCourse(course.id)} class="bg-yellow-500 hover:bg-yellow-700 text-white font-bold py-1 px-2 rounded mr-2">
                Update
              </button>
              <button on:click={() => deleteCourse(course.id)} class="bg-red-500 hover:bg-red-700 text-white font-bold py-1 px-2 rounded">
                Delete
              </button>
            </div>
          </div>
        {/each}
      </div>
  
      <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
        <h2 class="text-2xl font-bold mb-4">Manage Holidays</h2>
        <form on:submit|preventDefault={addHoliday} class="mb-4">
          <div class="grid grid-cols-2 gap-4">
            <div>
              <label for="month" class="block text-gray-700 text-sm font-bold mb-2">Month:</label>
              <select
                id="month"
                bind:value={newHoliday.month}
                class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
              >
                {#each months as month, index}
                  <option value={index + 1}>{month}</option>
                {/each}
              </select>
            </div>
            <div>
              <label for="days" class="block text-gray-700 text-sm font-bold mb-2">Days (comma-separated):</label>
              <input type="text" id="days" bind:value={newHoliday.days} required placeholder="1, 2, 3" class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
            </div>
          </div>
          <div class="mt-4">
            <button type="submit" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline">
              Add Holiday
            </button>
          </div>
        </form>
  
        <h3 class="text-xl font-bold mb-2">Existing Holidays:</h3>
        {#each holidays as holiday (holiday.id)}
          <div class="mb-4 p-4 border rounded">
            <p>{months[holiday.month - 1]}: {holiday.days.join(', ')}</p>
            <button on:click={() => deleteHoliday(holiday.id)} class="bg-red-500 hover:bg-red-700 text-white font-bold py-1 px-2 rounded mt-2">
              Delete
            </button>
          </div>
        {/each}
      </div>
    </main>
  {/if}