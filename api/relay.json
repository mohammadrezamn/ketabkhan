export const config = {
  runtime: "edge",
};

// --- تنظیمات اصلی (بدون تغییر منطق) ---
const TARGET = (process.env.TARGET_DOMAIN || "").replace(/\/$/, "");

const BLOCKED_HEADERS = new Set([
  "host", "connection", "keep-alive", "proxy-authenticate",
  "proxy-authorization", "te", "trailer", "transfer-encoding",
  "upgrade", "forwarded", "x-forwarded-host",
  "x-forwarded-proto", "x-forwarded-port"
]);

/**
 * Ketabkhan Relay Handler
 * تمام درخواست‌های ورودی روی /relay(.*) را دریافت و به TARGET منتقل می‌کند
 */
export default async function handler(req) {
  // بررسی وجود دامنه مقصد
  if (!TARGET) {
    return new Response("❌ TARGET_DOMAIN تنظیم نشده است", {
      status: 500,
      headers: { "content-type": "text/plain;charset=utf-8" }
    });
  }

  try {
    const url = new URL(req.url);

    // حذف پیشوند /relay از مسیر (اگر مستقیم صدا زده بشه)
    const cleanedPath = url.pathname.replace(/^\/relay/, "") || "/";
    const targetUrl = TARGET + cleanedPath + url.search;

    // فوروارد کردن هدرهای مجاز
    const headers = new Headers();
    let clientIp = null;

    for (const [key, value] of req.headers) {
      const k = key.toLowerCase();
      if (BLOCKED_HEADERS.has(k)) continue;
      if (k.startsWith("x-vercel-")) continue;

      if (k === "x-real-ip") {
        clientIp = value;
        continue;
      }
      if (k === "x-forwarded-for") {
        if (!clientIp) clientIp = value;
        continue;
      }

      headers.set(k, value);
    }

    if (clientIp) headers.set("x-forwarded-for", clientIp);

    // بدنهٔ درخواست (فقط برای متدهای غیر GET/HEAD)
    const method = req.method;
    const fetchOptions = {
      method,
      headers,
      redirect: "manual",
    };

    if (method !== "GET" && method !== "HEAD") {
      fetchOptions.body = req.body;
      fetchOptions.duplex = "half";
    }

    // ارسال درخواست به سرور مقصد
    const upstream = await fetch(targetUrl, fetchOptions);

    // ساخت پاسخ نهایی (انتقال هدرها و بدنه)
    const responseHeaders = new Headers();
    for (const [k, v] of upstream.headers) {
      if (k.toLowerCase() === "transfer-encoding") continue;
      responseHeaders.set(k, v);
    }

    return new Response(upstream.body, {
      status: upstream.status,
      headers: responseHeaders,
    });

  } catch (err) {
    console.error("Relay error:", err);
    return new Response("⚠️ خطا در تونل relay", {
      status: 502,
      headers: { "content-type": "text/plain;charset=utf-8" }
    });
  }
}
