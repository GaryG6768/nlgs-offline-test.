/* NLGS test app offline shell cache. Test repository only. */
const NLGS_CACHE = 'nlgs-offline-shell-v1';
const NLGS_SHELL = ['./', './index.html'];

self.addEventListener('install', event => {
  event.waitUntil((async () => {
    const cache = await caches.open(NLGS_CACHE);
    await Promise.all(NLGS_SHELL.map(async url => {
      try { await cache.add(url); } catch (e) { /* retry during later page loads */ }
    }));
    await self.skipWaiting();
  })());
});

self.addEventListener('activate', event => {
  event.waitUntil((async () => {
    const keys = await caches.keys();
    await Promise.all(keys.filter(key => key.startsWith('nlgs-offline-shell-') && key !== NLGS_CACHE).map(key => caches.delete(key)));
    await self.clients.claim();
  })());
});

self.addEventListener('fetch', event => {
  const request = event.request;
  if (request.method !== 'GET') return;
  const url = new URL(request.url);

  // Cache the app shell and same-origin app files. Network-first while online
  // keeps the test app fresh; cached responses are used when offline.
  if (url.origin === self.location.origin) {
    event.respondWith((async () => {
      const cache = await caches.open(NLGS_CACHE);
      try {
        const response = await fetch(request);
        if (response && response.ok) cache.put(request, response.clone()).catch(() => {});
        return response;
      } catch (error) {
        const cached = await cache.match(request) || await cache.match('./index.html') || await cache.match('./');
        if (cached) return cached;
        throw error;
      }
    })());
    return;
  }

  // Cache the Supabase JS CDN dependency after it has been loaded online.
  // API/data requests are not cached by this handler.
  if (url.hostname === 'cdn.jsdelivr.net') {
    event.respondWith((async () => {
      const cache = await caches.open(NLGS_CACHE);
      try {
        const response = await fetch(request);
        if (response && response.ok) cache.put(request, response.clone()).catch(() => {});
        return response;
      } catch (error) {
        const cached = await cache.match(request);
        if (cached) return cached;
        throw error;
      }
    })());
  }
});
