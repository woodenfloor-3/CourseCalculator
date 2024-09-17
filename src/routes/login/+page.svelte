<script>
    import { auth } from '$lib/firebase';
    import { signInWithEmailAndPassword } from 'firebase/auth';
    import { goto } from '$app/navigation';
  
    let email = '';
    let password = '';
    let error = null;
  
    async function handleLogin() {
      try {
        await signInWithEmailAndPassword(auth, email, password);
        goto('/admin');
      } catch (err) {
        error = err.message;
      }
    }
  </script>
  
  <main class="container mx-auto p-4 max-w-md">
    <h1 class="text-3xl font-bold mb-6 text-center text-blue-600">Login</h1>
  
    <form on:submit|preventDefault={handleLogin} class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
      <div class="mb-4">
        <label for="email" class="block text-gray-700 text-sm font-bold mb-2">Email:</label>
        <input type="email" id="email" bind:value={email} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
      </div>
  
      <div class="mb-6">
        <label for="password" class="block text-gray-700 text-sm font-bold mb-2">Password:</label>
        <input type="password" id="password" bind:value={password} required class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline">
      </div>
  
      {#if error}
        <p class="text-red-500 text-xs italic mb-4">{error}</p>
      {/if}
  
      <div class="flex items-center justify-center">
        <button type="submit" class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline">
          Login
        </button>
      </div>
    </form>
  </main>