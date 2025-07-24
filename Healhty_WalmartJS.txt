const OPENAI_MODEL = "gpt-4o-mini"; // change if you like

// ---------- Local state ----------
const els = {
  apiPanel: document.getElementById("api-panel"),
  toggleApi: document.getElementById("toggle-api"),
  apiKey: document.getElementById("apiKey"),
  saveKey: document.getElementById("saveKey"),
  goalsForm: document.getElementById("goals-form"),
  resetGoals: document.getElementById("resetGoals"),
  analyzeForm: document.getElementById("analyze-form"),
  url: document.getElementById("url"),
  results: document.getElementById("results"),
  demoBtn: document.getElementById("demoBtn"),
  productTpl: document.getElementById("product-card"),
};

// ---------- Init ----------
init();

function init() {
  els.toggleApi.addEventListener("click", () => {
    els.apiPanel.classList.toggle("hidden");
  });

  els.saveKey.addEventListener("click", () => {
    localStorage.setItem("OPENAI_API_KEY", els.apiKey.value.trim());
    alert("Saved.");
  });

  // load key
  els.apiKey.value = localStorage.getItem("OPENAI_API_KEY") || "";

  // load & save goals
  const savedGoals = loadGoals();
  fillGoalsForm(savedGoals);
  els.goalsForm.addEventListener("submit", (e) => {
    e.preventDefault();
    const goals = getGoalsFromForm();
    saveGoals(goals);
    alert("Goals saved.");
  });
  els.resetGoals.addEventListener("click", () => {
    localStorage.removeItem("health_goals");
    fillGoalsForm(defaultGoals());
  });

  // analyze
  els.analyzeForm.addEventListener("submit", async (e) => {
    e.preventDefault();
    clearResults();
    const url = els.url.value.trim();
    const goals = getGoalsFromForm();
    const fulfillment = goals.fulfillment;
    const apiKey = localStorage.getItem("OPENAI_API_KEY");

    showLoading("Working…");

    try {
      const items = apiKey
        ? await fetchAlternativesWithAI(apiKey, url, goals, fulfillment)
        : await demoData(url, goals);

      const scored = scoreAndSort(items, goals);
      renderResults(scored, goals);
    } catch (err) {
      console.error(err);
      showError(err?.message || "Something went wrong.");
    }
  });

  els.demoBtn.addEventListener("click", async () => {
    clearResults();
    showLoading("Loading demo…");
    const goals = getGoalsFromForm();
    const data = await demoData(
      "https://www.walmart.com/ip/Great-Value-Chocolate-Ice-Cream/123456",
      goals
    );
    const scored = scoreAndSort(data, goals);
    renderResults(scored, goals);
  });
}

// ---------- Goals ----------
function defaultGoals() {
  return {
    caloriesMax: 300,
    proteinMin: 20,
    sugarMax: 10,
    satFatMax: 5,
    sodiumMax: 600,
    fulfillment: "any",
  };
}
function loadGoals() {
  try {
    return JSON.parse(localStorage.getItem("health_goals")) || defaultGoals();
  } catch {
    return defaultGoals();
  }
}
function saveGoals(goals) {
  localStorage.setItem("health_goals", JSON.stringify(goals));
}
function fillGoalsForm(goals) {
  for (const [k, v] of Object.entries(goals)) {
    const el = els.goalsForm.elements[k];
    if (el) el.value = v;
  }
}
function getGoalsFromForm() {
  const f = new FormData(els.goalsForm);
  return {
    caloriesMax: +f.get("caloriesMax"),
    proteinMin: +f.get("proteinMin"),
    sugarMax: +f.get("sugarMax"),
    satFatMax: +f.get("satFatMax"),
    sodiumMax: +f.get("sodiumMax"),
    fulfillment: f.get("fulfillment"),
  };
}

// ---------- Fetch (AI or demo) ----------
async function fetchAlternativesWithAI(apiKey, url, goals, fulfillment) {
  const prompt = buildPrompt(url, goals, fulfillment);
  const res = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: OPENAI_MODEL,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content:
            "You are a nutrition- and shopping-assistant. Return strict JSON only.",
        },
        { role: "user", content: prompt },
      ],
      temperature: 0.3,
    }),
  });
  if (!res.ok) {
    const t = await res.text();
    throw new Error(`OpenAI error: ${t}`);
  }
  const data = await res.json();
  const json = safeParseJSON(data.choices?.[0]?.message?.content);
  if (!json?.alternatives) throw new Error("Bad AI response.");
  return json.alternatives.map(normalizeItem);
}

function buildPrompt(url, goals, fulfillment) {
  return `
Given this Walmart product URL:
${url}

And these user goals:
${JSON.stringify(goals, null, 2)}

Return up to 12 healthier Walmart alternatives that better satisfy the goals.
Prefer the fulfillment method "${fulfillment}" if possible (but it's okay if you cannot determine it).

Respond ONLY as valid JSON with this shape:

{
  "alternatives": [
    {
      "name": "string",
      "url": "https://www.walmart.com/...",
      "brand": "string",
      "image": "https://...",
      "price": 0,
      "fulfillment": "pickup|delivery|shipping|unknown",
      "nutrition": {
        "calories": 0,
        "protein_g": 0,
        "sugar_g": 0,
        "sat_fat_g": 0,
        "sodium_mg": 0
      }
    }
  ]
}

Do not add commentary.
`;
}

function normalizeItem(it) {
  return {
    name: it.name,
    url: it.url,
    brand: it.brand || "",
    image: it.image || "",
    price: typeof it.price === "number" ? it.price : null,
    fulfillment: it.fulfillment || "unknown",
    nutrition: {
      calories: num(it?.nutrition?.calories),
      protein_g: num(it?.nutrition?.protein_g),
      sugar_g: num(it?.nutrition?.sugar_g),
      sat_fat_g: num(it?.nutrition?.sat_fat_g),
      sodium_mg: num(it?.nutrition?.sodium_mg),
    },
  };
}
function num(v) {
  const n = Number(v);
  return Number.isFinite(n) ? n : null;
}

// Demo fallback
async function demoData(url, goals) {
  // Fake a short delay
  await new Promise((r) => setTimeout(r, 500));
  return [
    {
      name: "Fairlife Core Power Protein Shake, Vanilla",
      url: "https://www.walmart.com/ip/123456789",
      brand: "Fairlife",
      image:
        "https://i5.walmartimages.com/asr/placeholder.png",
      price: 7.48,
      fulfillment: "delivery",
      nutrition: {
        calories: 170,
        protein_g: 26,
        sugar_g: 5,
        sat_fat_g: 1.5,
        sodium_mg: 160,
      },
    },
    {
      name: "Great Value Greek Nonfat Yogurt, Plain",
      url: "https://www.walmart.com/ip/234567890",
      brand: "Great Value",
      image:
        "https://i5.walmartimages.com/asr/placeholder.png",
      price: 3.97,
      fulfillment: "pickup",
      nutrition: {
        calories: 90,
        protein_g: 17,
        sugar_g: 4,
        sat_fat_g: 0,
        sodium_mg: 55,
      },
    },
    {
      name: "Kodiak Power Cakes Flapjack & Waffle Mix, Buttermilk",
      url: "https://www.walmart.com/ip/345678901",
      brand: "Kodiak",
      image:
        "https://i5.walmartimages.com/asr/placeholder.png",
      price: 5.94,
      fulfillment: "shipping",
      nutrition: {
        calories: 190,
        protein_g: 14,
        sugar_g: 3,
        sat_fat_g: 0.5,
        sodium_mg: 380,
      },
    },
  ];
}

// ---------- Scoring ----------
function scoreAndSort(items, goals) {
  return items
    .map((it) => {
      const score = healthScore(it.nutrition, goals);
      return { ...it, _score: score };
    })
    .sort((a, b) => b._score - a._score);
}
function healthScore(n, g) {
  // Simple 0–100 score with penalties for exceeding maxes and bonuses for protein.
  if (!n) return 0;
  let score = 100;

  if (n.calories != null && g.caloriesMax > 0)
    score -= penalty(n.calories, g.caloriesMax);
  if (n.sugar_g != null && g.sugarMax >= 0)
    score -= penalty(n.sugar_g, g.sugarMax);
  if (n.sat_fat_g != null && g.satFatMax >= 0)
    score -= penalty(n.sat_fat_g, g.satFatMax);
  if (n.sodium_mg != null && g.sodiumMax >= 0)
    score -= penalty(n.sodium_mg, g.sodiumMax);

  if (n.protein_g != null && g.proteinMin > 0) {
    const diff = n.protein_g - g.proteinMin;
    if (diff >= 0) score += Math.min(15, diff * 1.5);
    else score -= Math.min(20, Math.abs(diff) * 2);
  }

  return Math.max(0, Math.min(100, Math.round(score)));
}
function penalty(value, maxAllowed) {
  if (maxAllowed <= 0) return 0;
  const over = value - maxAllowed;
  return over > 0 ? Math.min(25, over * 1.5) : 0;
}

// ---------- UI ----------
function clearResults() {
  els.results.innerHTML = "";
}
function showLoading(msg) {
  els.results.innerHTML = `<div class="card">${msg}</div>`;
}
function showError(msg) {
  els.results.innerHTML = `<div class="card" style="border-left:4px solid #e11d48;">${msg}</div>`;
}
function renderResults(items, goals) {
  clearResults();
  if (!items.length) {
    showError("No alternatives found.");
    return;
  }
  const frag = document.createDocumentFragment();
  items.forEach((item) => frag.appendChild(renderCard(item)));
  els.results.appendChild(frag);
}
function renderCard(item) {
  const tpl = els.productTpl.content.cloneNode(true);
  const root = tpl.querySelector(".product");
  root.querySelector(".score").textContent = item._score ?? "?";
  root.querySelector(".title").textContent = item.name;
  root.querySelector(".meta").innerHTML = `
    ${item.brand ? `<span class="badge">${item.brand}</span>` : ""}
    ${item.price != null ? `<span class="badge">$${item.price.toFixed(2)}</span>` : ""}
    ${item.fulfillment ? `<span class="badge">${item.fulfillment}</span>` : ""}
  `;
  const ul = root.querySelector(".nutrition");
  const n = item.nutrition || {};
  const fields = [
    ["Calories", n.calories, ""],
    ["Protein", n.protein_g, "g"],
    ["Sugar", n.sugar_g, "g"],
    ["Sat. Fat", n.sat_fat_g, "g"],
    ["Sodium", n.sodium_mg, "mg"],
  ];
  fields.forEach(([label, val, unit]) => {
    if (val == null) return;
    const li = document.createElement("li");
    li.textContent = `${label}: ${val}${unit}`;
    ul.appendChild(li);
  });
  const link = root.querySelector(".link");
  link.href = item.url;
  return tpl;
}

// ---------- Utils ----------
function safeParseJSON(str) {
  try {
    return JSON.parse(str);
  } catch {
    return null;
  }
}
