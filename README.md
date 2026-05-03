# Precision Detailing Dallas 🚗✨

Auto detailing booking website with live Google Calendar integration.

---

## 📁 File Structure

```
precision-detailing/
├── index.html          ← SEO meta tags + page shell
├── package.json        ← Project dependencies
├── vite.config.js      ← Build config
├── api/
│   └── claude.js       ← Serverless API proxy (keeps your API key secret)
└── src/
    ├── main.jsx        ← React entry point
    └── App.jsx         ← Full app (menu + booking)
```

---

## 🚀 Deploy in 4 Steps

### Step 1 — Push to GitHub
1. Go to [github.com](https://github.com) → click **New repository**
2. Name it `precision-detailing` → click **Create repository**
3. Upload all these files (drag & drop into the GitHub file browser)

### Step 2 — Deploy on Vercel (free)
1. Go to [vercel.com](https://vercel.com) → sign in with GitHub
2. Click **Add New Project** → select your `precision-detailing` repo
3. Click **Deploy** (Vercel auto-detects Vite)

### Step 3 — Add your API key (required for booking to work)
1. In Vercel → go to your project → **Settings** → **Environment Variables**
2. Add: `ANTHROPIC_API_KEY` = your key from [console.anthropic.com](https://console.anthropic.com)
3. Click **Save** → go to **Deployments** → **Redeploy**

### Step 4 — Get on Google (SEO)
1. Go to [Google Search Console](https://search.google.com/search-console)
2. Add your Vercel URL (e.g. `https://precision-detailing.vercel.app`)
3. Verify ownership → click **Request Indexing**
4. Also submit to [Bing Webmaster Tools](https://www.bing.com/webmasters)

---

## 🌐 Custom Domain (optional but recommended)

1. Buy `precisiondetailingdallas.com` on [Namecheap](https://namecheap.com) (~$10/yr)
2. In Vercel → **Settings** → **Domains** → add your domain
3. Follow Vercel's DNS instructions (takes ~10 min)

---

## 📞 Contact

Phone: (929) 476-1704  
Location: Dallas, TX
