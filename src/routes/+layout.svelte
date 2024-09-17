<script>
    import { onMount } from 'svelte';
    import { auth } from '$lib/firebase';
    import { goto } from '$app/navigation';
    import { page } from '$app/stores';
    import '../app.css'; // Adjust the path as needed
  
    let user = null;
  
    onMount(() => {
      const unsubscribe = auth.onAuthStateChanged((currentUser) => {
        user = currentUser;
        
        // If on admin page and not logged in, redirect to login
        if ($page.url.pathname === '/admin' && !user) {
          goto('/login');
        }
        
        // If on login page and already logged in, redirect to admin
        if ($page.url.pathname === '/login' && user) {
          goto('/admin');
        }
      });
  
      return unsubscribe;
    });
  </script>
  
  
  <slot />