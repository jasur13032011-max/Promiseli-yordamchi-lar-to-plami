# Promiseli-yordamchi-lar-to-plami
Asinxron dasturlashda murakkab jarayonlarni boshqarish, tarmoq xatolariga qarshi qayta urinish (retry) yoki parallel so'rovlar sonini cheklash uchun mo'ljallangan asinxron utilitlar moduli:

🛠️ Asinxron Utilitlar Moduli (asyncUtils.js)
JavaScript
// 1. Kutish (Delay) — Berilgan millisekund davomida kutuvchi sof Promise
export const kutish = (ms) => new Promise(resolve => setTimeout(resolve, ms));

// 2. Retry — Xatolik yuz berganda funksiyani qayta urinib ko'rish (Recursion bilan)
export const retry = async (fn, marotaba = 3, kechikish = 1000) => {
  try {
    return await fn();
  } catch (error) {
    if (marotaba <= 1) throw error; // Urinishlar tugasa, xatoni tashqariga otadi
    await kutish(kechikish);
    return retry(fn, marotaba - 1, kechikish); // Qayta urinish
  }
};

// 3. WithTimeout — Promise'ga bajarilish muddati cheklovini qo'shish (Promise.race ishlatilgan)
export const withTimeout = (promise, ms) => {
  const timeoutPromise = new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Vaqt tugadi (Timeout)")), ms)
  );
  return Promise.race([promise, timeoutPromise]);
};

// 4. Promisify — Eski uslubdagi error-first callback funksiyani Promise'ga o'tkazish
export const promisify = (callbackFn) => {
  return (...args) => new Promise((resolve, reject) => {
    callbackFn(...args, (error, result) => {
      if (error) reject(error);
      else resolve(result);
    });
  });
};

// 5. Parallel Limit — Bir vaqtning o'zida parallel bajariladigan topshiriqlar sonini cheklash
export const parallelLimit = async (tasks = [], limit = 2) => {
  const results = [];
  const executing = new Set();

  for (const task of tasks) {
    // Har bir topshiriqni Promise holatiga keltirib ishga tushiramiz
    const p = Promise.resolve().then(() => task());
    results.push(p);
    executing.add(p);

    // Topshiriq tugagach, uni bajarilayotganlar ro'yxatidan o'chiramiz
    const clean = () => executing.delete(p);
    p.then(clean, clean);

    // Agar limitga etgan bo'lsak, bajarilayotganlardan biri tugashini kutamiz (Promise.race)
    if (executing.size >= limit) {
      await Promise.race(executing);
    }
  }

  // Barcha natijalarni muvaffaqiyatli yoki xatosi bilan yig'ish (Promise.allSettled)
  return Promise.allSettled(results);
};
🎮 Demo Testlar va Xato Boshqaruvi (app.js)
JavaScript
import { kutish, retry, withTimeout, promisify, parallelLimit } from './asyncUtils.js';

// === DEMO 1: Retry Testi ===
const soxtaTarmoqSo'rovi = (() => {
  let urinish = 0;
  return async () => {
    urinish++;
    if (urinish < 3) throw new Error("Tarmoq xatosi (500)");
    return "Muvaffaqiyatli ma'lumot keldi!";
  };
})();

// === DEMO 2: Promisify uchun eski API ===
const eskiFaylTizimiAPI = (faylNomi, callback) => {
  setTimeout(() => {
    if (faylNomi === "xavfli.txt") callback(new Error("Ruxsat yo'q"), null);
    else callback(null, `Fayl tarkibi: ${faylNomi}`);
  }, 500);
};
const oqishPromise = promisify(eskiFaylTizimiAPI);

// ==========================================
// 🚀 ASOSIY AMALIY IJRO (ASYNC/AWAIT)
// ==========================================
const runDemo = async () => {
  console.log("=== 1. RETRY TESTI ===");
  try {
    const data = await retry(soxtaTarmoqSo'rovi, 4, 500);
    console.log("Retry Natijasi:", data); // 3-urinishda muvaffaqiyatli bo'ladi
  } catch (err) {
    console.error("Retry butunlay muvaffaqiyatsiz:", err.message);
  }

  console.log("\n=== 2. TIMEOUT TESTI (Xavfsiz holat) ===");
  try {
    const tezAmal = kutish(200).then(() => "Tezkor ma'lumot");
    const natija = await withTimeout(tezAmal, 500);
    console.log("Timeout Natijasi (Muvaffaqiyat):", natija);
  } catch (err) {
    console.error("Timeout Xatosi:", err.message);
  }

  console.log("\n=== 3. TIMEOUT TESTI (Xatolik holati) ===");
  try {
    const sekinAmal = kutish(1000).then(() => "Sekin ma'lumot");
    await withTimeout(sekinAmal, 400); // 400ms da uziladi
  } catch (err) {
    console.log("Kutilgan xatolik yuz berdi:", err.message); // Vaqt tugadi (Timeout)
  }

  console.log("\n=== 4. PROMISIFY TESTI ===");
  try {
    const faylMatni = await oqishPromise("data.json");
    console.log("Promisify Natijasi:", faylMatni);
  } catch (err) {
    console.error("Promisify Xatosi:", err.message);
  }

  console.log("\n=== 5. PARALLEL LIMIT TESTI ===");
  // 5 ta og'ir vazifa, lekin bir vaqtda faqat 2 tasi ishlaydi
  const vazifalar = [
    () => kutish(300).then(() => "Vazifa 1"),
    () => kutish(100).then(() => "Vazifa 2"),
    () => kutish(400).then(() => { throw new Error("Vazifa 3 sindi"); }),
    () => kutish(200).then(() => "Vazifa 4"),
    () => kutish(150).then(() => "Vazifa 5")
  ];

  console.time("ParallelLimit Vaqti");
  const yakuniyNatijalar = await parallelLimit(vazifalar, 2);
  console.timeEnd("ParallelLimit Vaqti");

  console.log("Barcha parallel topshiriqlar holati:");
  yakuniyNatijalar.forEach((res, index) => {
    if (res.status === "fulfilled") {
      console.log(`-> [${index + 1}] Muvaffaqiyat: ${res.value}`);
    } else {
      console.log(`-> [${index + 1}] Xato: ${res.reason.message}`);
    }
  });
};

runDemo();
