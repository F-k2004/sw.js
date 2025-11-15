// sw.j
const CACHE_NAME = 'pwa-weather-static-v1';
const DATA_CACHE = 'pwa-weather-data-v1';

const STATIC_ASSETS = [
  '/', // index.html
  '/index.html',
  '/app.js',
  '/manifest.json',
  // اگر آیکون‌ها یا CSS جدا داری اینجا اضافه کن:
  '/icons/icon-192.png',
  '/icons/icon-512.png'
];

// نصب و کش استاتیک
self.addEventListener('install', evt => {
  evt.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_ASSETS))
      .then(() => self.skipWaiting())
  );
});

// فعالسازی و پاکسازی کش‌های قدیمی
self.addEventListener('activate', evt => {
  evt.waitUntil(
    caches.keys().then(keys => Promise.all(
      keys.map(key => {
        if (key !== CACHE_NAME && key !== DATA_CACHE) return caches.delete(key);
      })
    ))
  );
  self.clients.claim();
});

// استراتژی fetch:
// - برای درخواست‌های API آب‌وهوا: network first -> fallback to cache
// - برای بقیه استاتیک‌ها: cache first
self.addEventListener('fetch', evt => {
  const req = evt.request;
  const url = new URL(req.url);

  // مثال: درخواست به openweathermap
  if (url.hostname.includes('api.openweathermap.org')) {
    evt.respondWith(
      caches.open(DATA_CACHE).then(cache =>
        fetch(req).then(response => {
          // کلون و ذخیره پاسخ در کش برای استفاده آفلاین
          if (response.status === 200) cache.put(req, response.clone());
          return response;
        }).catch(() =>
          cache.match(req)
        )
      )
    );
    return;
  }

  // سایر منابع استاتیک
  evt.respondWith(
    caches.match(req).then(cached => cached || fetch(req).then(res => {
      // می‌توانیم منابع جدید را به کش عمومی هم اضافه کنیم (اختیاری)
      return res;
    })).catch(() => {
      // برای صفحات navigation یا index fallback ساده
      if (req.mode === 'navigate') {
        return caches.match('/index.html');
      }
    })
  );
});
