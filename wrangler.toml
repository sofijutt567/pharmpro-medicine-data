/**
 * PharmPro Medicines + User Stock Worker
 * ---------------------------------------
 * MEDICINES_KV   -> master medicines list (key: "medicines_db") - shared, read-only, 1000 items
 * USER_STOCK_KV  -> per-user overrides + custom additions, key = "user_<uid>"
 *
 * wrangler.toml (2 KV bindings needed now):
 *   [[kv_namespaces]]
 *   binding = "MEDICINES_KV"
 *   id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
 *
 *   [[kv_namespaces]]
 *   binding = "USER_STOCK_KV"
 *   id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyy"
 *
 * Upload master data once (same as before):
 *   wrangler kv:key put --binding=MEDICINES_KV "medicines_db" --path=medicinesdata.js
 *
 * Endpoints:
 *   GET  /search?q=...&category=...   -> master medicines search (unchanged)
 *   GET  /all                          -> full master list (unchanged)
 *   GET  /user?uid=XXX                 -> { overrides, additions } for that user
 *   POST /user/save                    -> upsert master-item override OR custom addition
 *   POST /user/flags                   -> update alert flags only (stock/expiry email system)
 *   POST /user/delete                  -> delete a custom addition (master data can NEVER be deleted)
 */

// In-memory cache for master list (resets when Worker instance recycles)
let cachedMedicines = null;
let cachedAt = 0;
const CACHE_TTL_MS = 5 * 60 * 1000; // 5 minutes

async function loadMedicines(env) {
    const now = Date.now();
    if (cachedMedicines && (now - cachedAt) < CACHE_TTL_MS) {
        return cachedMedicines;
    }
    const raw = await env.MEDICINES_KV.get("medicines_db");
    if (!raw) return [];
    const parsed = JSON.parse(raw);
    cachedMedicines = parsed;
    cachedAt = now;
    return parsed;
}

async function loadUserData(env, uid) {
    const raw = await env.USER_STOCK_KV.get(`user_${uid}`);
    if (!raw) return { overrides: {}, additions: [] };
    try {
        const parsed = JSON.parse(raw);
        return { overrides: parsed.overrides || {}, additions: parsed.additions || [] };
    } catch {
        return { overrides: {}, additions: [] };
    }
}

async function saveUserData(env, uid, data) {
    await env.USER_STOCK_KV.put(`user_${uid}`, JSON.stringify(data));
}

function jsonResponse(data, status = 200) {
    return new Response(JSON.stringify(data), {
        status,
        headers: {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
            "Access-Control-Allow-Headers": "Content-Type"
        }
    });
}

export default {
    async fetch(request, env) {
        const url = new URL(request.url);

        // CORS preflight
        if (request.method === "OPTIONS") {
            return jsonResponse({}, 204);
        }

        // ---- Master medicines list/search (unchanged from before) ----
        if (url.pathname === "/all") {
            const medicines = await loadMedicines(env);
            return jsonResponse({ count: medicines.length, results: medicines });
        }

        if (url.pathname === "/search") {
            const q = (url.searchParams.get("q") || "").trim().toLowerCase();
            const categoryFilter = (url.searchParams.get("category") || "").trim().toLowerCase();
            const medicines = await loadMedicines(env);

            let results = medicines;
            if (categoryFilter) {
                results = results.filter(m => (m.category || "").toLowerCase() === categoryFilter);
            }
            if (q) {
                results = results.filter(m =>
                    (m.name || "").toLowerCase().includes(q) ||
                    (m.generic || "").toLowerCase().includes(q) ||
                    (m.category || "").toLowerCase().includes(q)
                );
            }
            results = results.slice(0, 25);
            return jsonResponse({ count: results.length, results });
        }

        // ---- Per-user overrides / custom additions ----
        if (url.pathname === "/user" && request.method === "GET") {
            const uid = url.searchParams.get("uid");
            if (!uid) return jsonResponse({ error: "uid required" }, 400);
            const data = await loadUserData(env, uid);
            return jsonResponse(data);
        }

        if (url.pathname === "/user/save" && request.method === "POST") {
            let body;
            try { body = await request.json(); } catch { return jsonResponse({ error: "Invalid JSON" }, 400); }

            const { uid, id, isMasterId, name, power, qty, price, expiry, category, generic, company, status, formType } = body;
            if (!uid) return jsonResponse({ error: "uid required" }, 400);
            if (!name && !id) return jsonResponse({ error: "name required" }, 400);

            const data = await loadUserData(env, uid);

            if (id && isMasterId) {
                // Master medicine ke liye is user ka apna override
                // (status/formType qty se independent flags hain, isliye yahan bhi save hote hain)
                data.overrides[id] = {
                    ...(data.overrides[id] || {}),
                    power, qty, price, expiry,
                    status: status || 'in_stock',
                    formType: formType || '',
                    updatedAt: Date.now()
                };
            } else if (id && !isMasterId) {
                // Pehle se maujood custom addition update ho rahi hai
                const idx = data.additions.findIndex(a => a.id === id);
                if (idx >= 0) {
                    data.additions[idx] = { ...data.additions[idx], name, power, qty, price, expiry, status: status || 'in_stock', formType: formType || '' };
                } else {
                    data.additions.push({
                        id, name, power, qty, price, expiry,
                        category: category || '', generic: generic || '', company: company || '',
                        status: status || 'in_stock', formType: formType || ''
                    });
                }
            } else {
                // Bilkul nayi custom medicine
                const newId = "c" + Date.now() + Math.random().toString(36).slice(2, 6);
                data.additions.push({
                    id: newId, name, power, qty, price, expiry,
                    category: category || '', generic: generic || '', company: company || '',
                    status: status || 'in_stock', formType: formType || ''
                });
            }

            await saveUserData(env, uid, data);
            return jsonResponse({ success: true });
        }

        if (url.pathname === "/user/flags" && request.method === "POST") {
            let body;
            try { body = await request.json(); } catch { return jsonResponse({ error: "Invalid JSON" }, 400); }

            const { uid, id, isMasterId, flags } = body;
            if (!uid || !id) return jsonResponse({ error: "uid and id required" }, 400);

            const data = await loadUserData(env, uid);
            if (isMasterId) {
                data.overrides[id] = { ...(data.overrides[id] || {}), ...flags };
            } else {
                const idx = data.additions.findIndex(a => a.id === id);
                if (idx >= 0) data.additions[idx] = { ...data.additions[idx], ...flags };
            }
            await saveUserData(env, uid, data);
            return jsonResponse({ success: true });
        }

        if (url.pathname === "/user/delete" && request.method === "POST") {
            let body;
            try { body = await request.json(); } catch { return jsonResponse({ error: "Invalid JSON" }, 400); }

            const { uid, id } = body;
            if (!uid || !id) return jsonResponse({ error: "uid and id required" }, 400);

            if (!String(id).startsWith("c")) {
                return jsonResponse({ error: "Master medicine data delete nahi ho sakti." }, 403);
            }

            const data = await loadUserData(env, uid);
            data.additions = data.additions.filter(a => a.id !== id);
            await saveUserData(env, uid, data);
            return jsonResponse({ success: true });
        }

        return jsonResponse({ error: "Not found. Use /search, /all, /user, /user/save, /user/flags, /user/delete" }, 404);
    }
};
