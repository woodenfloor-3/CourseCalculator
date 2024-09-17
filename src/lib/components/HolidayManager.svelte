<script>
    import { db } from '$lib/firebase';
    import { collection, addDoc, getDocs, deleteDoc, doc } from 'firebase/firestore';
    import { onMount } from 'svelte';
  
    let holidays = [];
    let newHoliday = { month: 1, days: '' };
  
    onMount(fetchHolidays);
  
    async function fetchHolidays() {
      const querySnapshot = await getDocs(collection(db, 'holidays'));
      holidays = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    }
  
    async function addHoliday() {
      if (!newHoliday.days) {
        alert('Please enter holiday days');
        return;
      }
      const holidayToAdd = {
        month: parseInt(newHoliday.month),
        days: newHoliday.days.split(',').map(day => parseInt(day.trim())).filter(day => !isNaN(day) && day > 0 && day <= 31)
      };
      await addDoc(collection(db, 'holidays'), holidayToAdd);
      newHoliday = { month: 1, days: '' };
      await fetchHolidays();
    }
  
    async function deleteHoliday(id) {
      await deleteDoc(doc(db, 'holidays', id));
      await fetchHolidays();
    }
  
    const months = [
      'January', 'February', 'March', 'April', 'May', 'June',
      'July', 'August', 'September', 'October', 'November', 'December'
    ];
  </script>
  
  <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
    <h2 class="text-2xl font-bold mb-4">Manage Holidays</h2>
    <form on:submit|preventDefault={addHoliday} class="mb-4">
      <div class="flex items-center space-x-4">
        <div class="flex-1">
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
        <div class="flex-1">
          <label for="days" class="block text-gray-700 text-sm font-bold mb-2">Days (comma-separated):</label>
          <input
            type="text"
            id="days"
            bind:value={newHoliday.days}
            placeholder="1, 2, 3"
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          >
        </div>
        <div class="flex-none">
          <button
            type="submit"
            class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
          >
            Add Holiday
          </button>
        </div>
      </div>
    </form>
  
    <div>
      <h3 class="text-xl font-bold mb-2">Current Holidays:</h3>
      {#each holidays as holiday (holiday.id)}
        <div class="flex items-center justify-between py-2 border-b">
          <span>{months[holiday.month - 1]}: {holiday.days.join(', ')}</span>
          <button
            on:click={() => deleteHoliday(holiday.id)}
            class="bg-red-500 hover:bg-red-700 text-white font-bold py-1 px-2 rounded focus:outline-none focus:shadow-outline"
          >
            Delete
          </button>
        </div>
      {/each}
    </div>
  </div>