# AIkidgenius
A child‑friendly AI‑powered learning app with parent/child auth, AI‑generated lessons, and subscription access.
Great! Here's the full workflow to get your AIkidgenius app live with one-click deployment to Vercel and GitHub—includes Firebase Auth, Firestore, GPT-4o lesson generation, and Stripe payment setup.


---

🚀 Step 1: GitHub Repository Setup

1. Create a new empty GitHub repo named AIkidgenius.

2. Add this README.md and supporting files.

# AIkidgenius

A child‑friendly AI‑powered learning app with parent/child auth, AI‑generated lessons, and subscription access.

## One-Click Deploy

Click below to deploy to Vercel:

[![Deploy to Vercel](https://vercel.com/button)](https://vercel.com/import/project?template=<YOUR_REPO_URL>)

---

## 🔧 Setup Instructions

### 1. Firebase
- Create a Firebase project.
- Enable Auth (Email sign-in).
- Create Firestore DB.
- Copy config to `.env`

NEXT_PUBLIC_FIREBASE_API_KEY=... NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=... NEXT_PUBLIC_FIREBASE_PROJECT_ID=...

### 2. OpenAI
- Create API key
- Add to `.env`

OPENAI_API_KEY=...

### 3. Stripe (or LemonSqueezy)
- Create product/pricing.
- Add secret and webhook signing key:

STRIPE_SECRET_KEY=... STRIPE_WEBHOOK_SECRET=...

### 4. Vercel Environment
Set the same `.env` values in Vercel.

### 5. Deploy
Push changes & follow Vercel deployment.  
App is live within minutes.

---

## 📦 Project Structure

AIkidgenius/ ├─ pages/ │  ├─ index.tsx           # Landing / login │  ├─ dashboard.tsx       # Age‑based lesson display │  └─ api/ │      ├─ generate.ts     # AI lesson generator │      └─ stripe-webhook.ts # Payment webhook ├─ lib/ │  ├─ firebase.ts │  └─ stripe.ts ├─ components/ │  ├─ LessonCard.tsx │  └─ PaymentButton.tsx ├─ cron.js                # Weekly content trigger ├─ vercel.json └─ package.json

---

### 🔄 Weekly Cron Job
Use Vercel cron or GitHub Actions to call `/api/generate` for new lessons.

0 0 * * 1 curl https://<YOUR_APP>/api/generate

---

### 📚 What You Get
- Firebase Auth with parent‑child profiles  
- GPT‑4o lesson generation via API  
- Stripe Subscription payment integration  
- Vercel-hosted web app  
- Cron‑driven weekly lessons


---

🔧 Step 2: Core Files

Here’s a quick look at key files:

lib/firebase.ts

import { initializeApp } from "firebase/app";
import { getAuth } from "firebase/auth";
import { getFirestore } from "firebase/firestore";

const app = initializeApp({
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
});

export const auth = getAuth(app);
export const db = getFirestore(app);


---

pages/api/generate.ts

import { NextApiRequest, NextApiResponse } from 'next'
import { OpenAI } from 'openai';
import { addDoc, collection } from 'firebase/firestore';
import { db } from '../../lib/firebase';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // TODO: restrict to cron or admin
  const ageGroup = req.query.ageGroup || '3-6';
  const prompt = `Generate a fun weekly lesson for kids age ${ageGroup}: include a story, quiz, and activity.`;
  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: 'system', content: 'You are a playful children's teacher.' },
               { role: 'user', content: prompt }]
  });
  const content = response.choices[0].message?.content;
  await addDoc(collection(db, 'lessons'), {
    ageGroup,
    content,
    createdAt: new Date().toISOString()
  });
  res.status(200).json({ ok: true });
}


---

pages/api/stripe-webhook.ts

import { buffer } from 'micro';
import Stripe from 'stripe';
import { stripe } from '../../lib/stripe';

export const config = { api: { bodyParser: false } };

export default async function handler(req, res) {
  const buf = await buffer(req);
  const sig = req.headers['stripe-signature']!;
  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(buf, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${(err as Error).message}`);
  }

  if (['checkout.session.completed'].includes(event.type)) {
    const session = event.data.object as any;
    // TODO: update user subscription in Firestore
  }

  res.json({ received: true });
}


---

pages/dashboard.tsx

import { useEffect, useState } from 'react';
import { auth, db } from '../lib/firebase';
import { collection, query, where, getDocs } from 'firebase/firestore';
import PaymentButton from '../components/PaymentButton';
import LessonCard from '../components/LessonCard';

export default function Dashboard() {
  const [lessons, setLessons] = useState([]);
  const [user] = useAuthState(auth);

  useEffect(() => {
    if (!user) return;
    const q = query(collection(db, 'lessons'), where('ageGroup', '==', user.ageGroup));
    getDocs(q).then(snap => setLessons(snap.docs.map(d => d.data())));
  }, [user]);

  if (!user) return <div>Please log in...</div>;

  return (
    <div>
      <h1>Welcome, {user.displayName}</h1>
      {!user.isPaid && <PaymentButton />}
      {user.isPaid && lessons.map((l, i) => <LessonCard key={i} content={l.content} />)}
    </div>
  );
}

