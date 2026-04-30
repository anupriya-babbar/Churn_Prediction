# glōw — Skincare Discovery Platform

A modern skincare e-commerce prototype built with HTML, CSS, and JavaScript.
Inspired by Tira Beauty & Nykaa's design language, tailored for Indian women aged 18–40.

## Features

- **Explore Products** — Browse 12+ products across 8 categories
- **Smart Filters** — Filter by skin type (oily, dry, combination, sensitive, normal)
- **Compare Tool** — Side-by-side comparison of up to 4 products
- **Skin Copilot** — AI-powered chat for personalised skincare recommendations
- **Shopping Cart** — Add, update quantities, and place orders
- **Product Detail** — Full page with rating breakdown and reviews
- **Community Reviews** — Submit and browse product feedback

## Project Structure

```
glow-skincare/
├── index.html        ← Main HTML (all pages)
├── css/
│   └── style.css     ← All styles and design tokens
├── js/
│   ├── data.js       ← Product catalogue data
│   └── app.js        ← All app logic and interactions
└── README.md
```

## Tech Stack (Prototype)

| Layer    | Technology              |
|----------|-------------------------|
| Frontend | HTML5, CSS3, Vanilla JS |
| Fonts    | Cormorant Garamond + DM Sans (Google Fonts) |

## Recommended Production Stack

| Layer     | Technology                        |
|-----------|-----------------------------------|
| Frontend  | React + Next.js                   |
| Backend   | Java Spring Boot (REST APIs)      |
| Database  | MongoDB (products) + Redis (cart) |
| LLM       | Gemini Flash 2.0 / Claude Haiku   |
| Search    | Elasticsearch                     |
| Auth      | Firebase Auth                     |
| Payments  | Razorpay                          |
| Hosting   | Vercel (FE) + AWS/GCP (BE)        |

## Getting Started

Just open `index.html` in any browser — no build step needed.

```bash
# Clone the repo
git clone https://github.com/anupriya-babbar/D2C-SkinCare-Website.git

# Open in browser
open index.html
```

## Live Preview

Deploy instantly on [Netlify Drop](https://app.netlify.com/drop) or [Vercel](https://vercel.com) by dragging the project folder.

---

Built with ♥ for the modern Indian skincare shopper.
