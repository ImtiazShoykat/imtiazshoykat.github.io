---
title: "SQL এর দুই ভাই! CTE vs Subquery — কখন কোনটা? গুলিয়ে ফেলবেন না! পার্ট ২"
date: 2026-10-11 11:00:00 +0600
categories: [SQL, Database]
tags: [cte, subquery, sql-interview]
---

# SQL এর দুই ভাই! CTE vs Subquery — কখন কোনটা? গুলিয়ে ফেলবেন না! পার্ট ২

আগের পর্বে আমরা থেমেছিলাম একটা প্রশ্ন নিয়ে — "এভারেজ থেকে কে বেশি খরচ করলো। করলো তো করলো, **কত বেশি করলো?**"

আপনি নিশ্চয়ই চেষ্টা করে দেখেছেন। পারেননি? চিন্তা নেই ভাই। আজকে সেটাই বের করবো। আর সেই সাথে দেখবো CTE-র আসল শক্তিটা — যেটা আগের পর্বে শুধু বলে গিয়েছিলাম, আজকে হাতে কলমে দেখাবো।

আগের পর্বে যারা miss করেছেন, তাদের জন্য এক লাইনে — Subquery হলো মেইন কোয়েরির ভেতরে লুকিয়ে থাকা কোয়েরি, আর CTE হলো WITH দিয়ে শুরু হওয়া আলাদা নাম দেওয়া কোয়েরি। দুটোর রেজাল্ট একই, শুধু পড়ার সুবিধা আর ব্যবহারের ধরন আলাদা।

---

## আমাদের ডাটা টেবিলটি ছিল এরকম 

| Name | City | Trip Day | Total Spent |
|------|----------|-----|-----|
| **Harry** | New York | 1 | $200 |
| **Johny** | New York | 2 | $190 |
| **Nicholas** | Las Vegas | 1 | $190 |
| **Bailey** | Las Vegas | 2 | $180 |
| **Harry** | Las Vegas | 3 | $190 |
---


## এবার আসল কাজে আসি

আগের কোয়েরিটা মনে আছে তো?

```sql
WITH average_spent AS (
    SELECT AVG(spent) AS avg_spent
    FROM trip_expenses
)
SELECT t.name, t.spent
FROM trip_expenses t
JOIN average_spent a
    ON t.spent > a.avg_spent;
```

এটা আমাদের বলেছিল — Harry এভারেজের চেয়ে বেশি খরচ করেছে। কিন্তু প্রশ্ন ছিল — **কত বেশি?**

এখন কী করবো? খুব সহজ। এভারেজ থেকে Harry-র খরচ বিয়োগ করলেই তো হয়ে গেল!

```sql
WITH average_spent AS (
    SELECT AVG(spent) AS avg_spent
    FROM trip_expenses
)
SELECT t.name, t.spent, a.avg_spent,
       (t.spent - a.avg_spent) AS extra_spent
FROM trip_expenses t
JOIN average_spent a
    ON t.spent > a.avg_spent;
```

দেখুন তো — `extra_spent` নামে একটা নতুন কলাম বেরিয়ে এলো। এটাই বলবে Harry এভারেজের চেয়ে কত বেশি খরচ করেছে। এত সহজ, তাই না?

---

## এখন আসি CTE-র আসল শক্তিতে

আগের পর্বে বলেছিলাম — CTE-র একটা বড় সুবিধা হলো, একবার নাম দিয়ে রাখলে বারবার ইউজ করা যায়। কিন্তু সেটা তো দেখানো হয়নি। আজকে সেটাই দেখি।

ধরুন, ইন্টারভিউয়ার আরও এক ধাপ এগিয়ে গেলেন। তিনি বললেন — "আচ্ছা, কত বেশি খরচ করলো সেটা বের করলেন। এখন সবচেয়ে বেশি খরচ করা থেকে কম খরচ করা পর্যন্ত সাজিয়ে দেখান।"

এখানেই CTE-র আসল মজা। আমরা একটার পর একটা ধাপ সাজাতে পারি — একটা CTE-র রেজাল্ট আরেকটা CTE-তে ইউজ করতে পারি।

```sql
WITH average_spent AS (
    SELECT AVG(spent) AS avg_spent
    FROM trip_expenses
),
above_average AS (
    SELECT t.name, t.spent, a.avg_spent,
           (t.spent - a.avg_spent) AS extra_spent
    FROM trip_expenses t
    JOIN average_spent a
        ON t.spent > a.avg_spent
)
SELECT name, spent, extra_spent
FROM above_average
ORDER BY extra_spent DESC;
```

খেয়াল করলেন তো? এখানে দুইটা CTE আছে — `average_spent` আর `above_average`। প্রথমটা এভারেজ বের করলো, দ্বিতীয়টা সেটা ইউজ করে বের করলো কত বেশি খরচ হয়েছে, আর শেষে মেইন কোয়েরি সেটাকে সাজিয়ে দিলো।

ভাবুন তো — এই একই কাজ যদি Subquery দিয়ে করতে হতো, কেমন হতো?

```sql
SELECT name, spent, (spent - avg_spent) AS extra_spent
FROM (
    SELECT t.name, t.spent, a.avg_spent
    FROM trip_expenses t
    JOIN (
        SELECT AVG(spent) AS avg_spent
        FROM trip_expenses
    ) a
    ON t.spent > a.avg_spent
)
ORDER BY extra_spent DESC;
```

দেখুন — সাবকুয়েরির ভেতরে সাবকুয়েরি, তার ভেতরে আবার সাবকুয়েরি। পড়তে গেলে চোখ ঘুরে যায়। কোনটা আগে চলছে, কোনটা পরে — বুঝতে সময় লাগে। CTE-তে সেটা ধাপে ধাপে সাজানো — একদম সিঁড়ি ভাঙার মতো।

**এটাই CTE-র আসল শক্তি ভাই। পারফরম্যান্স না — পড়ার সুবিধা আর ধাপে ধাপে সাজানোর ক্ষমতা।**

---

## একটা ছোট পরীক্ষা — পারফরম্যান্স আসলে কেমন?

আগের পর্বে বলেছিলাম — "CTE মানেই ফাস্ট না, Subquery মানেই স্লো না।" আজকে সেটার একটা ছোট প্রমাণ দেখি।

আপনি যদি ডেটাবেসে এই দুইটা কোয়েরি চালান, আর তার সামনে `EXPLAIN` বা `EXPLAIN ANALYZE` লিখে দেন — দেখবেন ডেটাবেস কীভাবে কাজ করছে:

```sql
EXPLAIN ANALYZE
WITH average_spent AS (
    SELECT AVG(spent) AS avg_spent
    FROM trip_expenses
)
SELECT t.name, t.spent
FROM trip_expenses t
JOIN average_spent a
    ON t.spent > a.avg_spent;
```

আর সাবকুয়েরি ভার্সন:

```sql
EXPLAIN ANALYZE
SELECT name, spent
FROM trip_expenses
WHERE spent > (
    SELECT AVG(spent)
    FROM trip_expenses
);
```

দুটোর আউটপুট পাশাপাশি রাখলে দেখবেন — কিছু ডেটাবেসে দুটো প্রায় একই, কিছুতে একটা আরেকটার চেয়ে দ্রুত। কোনটা দ্রুত সেটা নির্ভর করে টেবিলের সাইজ, ইনডেক্স, আর ডেটাবেসের উপর। তাই শুধু CTE দেখে "ফাস্ট" বলে ধরে নেওয়া যায় না — পরীক্ষা করে দেখতে হয়।

---

## তাহলে কখন কোনটা?

আগের পর্বের ৩টা প্রশ্নই যথেষ্ট। তবুও একবার ঝালিয়ে নিই:

- **একবার লাগবে, ছোট কাজ** → Subquery
- **বারবার লাগবে, বড় কাজ, ধাপে ধাপে সাজাতে হবে** → CTE
- **টিমে শেয়ার হবে, পরে ডিবাগ করতে হবে** → CTE

আর একটা কথা মনে রাখবেন — **কোনোটাই খারাপ না।** দুটোই SQL-এর হাতিয়ার। ঠিক যেমন ছুরি আর কাঁচি — দুটোই কাটে, কিন্তু কোনটা দিয়ে কী কাটবেন সেটা বুঝতে হয়।

---

👉 [পার্ট ১ পড়ুন](/sql-cte-vs-subquery/)
