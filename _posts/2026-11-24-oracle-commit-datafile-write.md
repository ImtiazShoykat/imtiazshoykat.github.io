# Oracle-এ COMMIT করলেই কি সাথে সাথেই Datafile-এ Data Write হয়?

Database-এ আপনি একটি টেবিলে ডাটা ইনসার্ট করে `COMMIT` দিলেন। মনে হচ্ছে ডাটা বুঝি সাথে সাথেই Datafile-এ সেভ হয়ে গেলো?

**আসলে না।** `COMMIT` করার সাথে সাথেই `DBWn` (Database Writer) ডাটা Datafile-এ লিখে ফেলে না।

তাহলে কখনো কি ভেবে দেখেছেন— Datafile-এ ডাটা না লিখেও Database কীভাবে নিশ্চিত করে যে আপনার Committed Data নিরাপদে আছে? কেনই বা `DBWn` সাথে সাথে ডাটা লিখে না?

> কোনো একদিন DBA Interview Board-এ আপনাকে প্রশ্ন করা হলো:
> *"Why doesn't DBWn write modified data blocks to datafiles immediately after every COMMIT?"*

তখন আপনার উত্তর কী হবে? বেশিরভাগ মানুষ এখানেই আটকে যান — কারণ আমরা জানি COMMIT মানে "save"। কিন্তু Oracle-এর দুনিয়ায় "save" শব্দটার মানে একটু আলাদা। চলুন, বিষয়টা একটু সহজভাবে বোঝার চেষ্টা করি।

## তাহলে COMMIT-এর সময় আসলে কী হয়?

ভাবুন তো — আপনি `COMMIT` চাপলেন। এই একটা শব্দের পেছনে Oracle-এর ভেতরে দুটো আলাদা জগৎ কাজ করছে।

প্রথম জগৎটা হলো **`LGWR`** (Log Writer)-এর। আপনি COMMIT করার সঙ্গে সঙ্গে `LGWR` তার Redo Information — অর্থাৎ "কী কী পরিবর্তন হলো" তার একটা ডায়েরি — Online Redo Log File-এ লিখে ফেলে। এই লেখাটা হলো **sequential** — একটার পর একটা, পেছনে পেছনে। অনেকটা ডায়েরিতে দিনের ঘটনা লিখে রাখার মতো।

দ্বিতীয় জগৎটা হলো **`DBWn`** (Database Writer)-এর। এর কাজ হলো Database Buffer Cache-এ জমে থাকা Dirty Blocks — মানে যেসব Block-এ পরিবর্তন হয়েছে — সেগুলো Datafile-এ লেখা। কিন্তু এই লেখাটা **random** — এ Block, ও Block, ভিন্ন Datafile, ভিন্ন জায়গা। ছড়িয়ে ছিটিয়ে।

এখন প্রশ্ন হলো — `LGWR` তো COMMIT-এর সময়ই Redo লিখে ফেলছে, তাহলে `DBWn` কেন একই সময়ে Datafile-এ Data Write করে না?

## কেন DBWn সাথে সাথে লেখে না?

একটি ছোট উদাহরণ দিই।

ধরুন, আপনি একটি রেস্টুরেন্টে কাজ করেন। প্রতিটি কাস্টমার খাওয়া শেষ করার পর যদি একটি করে প্লেট ধুতে যান, তাহলে আপনার অনেক সময় নষ্ট হবে। তার পরিবর্তে অনেকগুলো প্লেট একসঙ্গে সংগ্রহ করে Batch আকারে ধুলে কাজটি দ্রুত এবং সহজে করা সম্ভব।

Oracle-এর `DBWn`-ও অনেকটা একইভাবে কাজ করে। প্রতিটি COMMIT-এর পর আলাদাভাবে Datafile-এ Data Write না করে, প্রয়োজন অনুযায়ী একসঙ্গে অনেকগুলো Dirty Block Write করে।

কিন্তু এখানে একটা জিনিস খেয়াল করুন — প্লেট ধোয়া মানে random কাজ, আর ডায়েরি লেখা মানে sequential কাজ। sequential কাজ সবসময় দ্রুত এবং সস্তা। তাই Oracle-এর নিয়ম হলো: **আগে ডায়েরি (redo), পরে প্লেট (datafile)**। এতে অপ্রয়োজনীয় Disk I/O কমে এবং Database Performance ভালো থাকে।

## কিন্তু ধরুন, COMMIT-এর পরপরই Crash হলো!

এখন আরেকটি গুরুত্বপূর্ণ বিষয় চিন্তা করুন।

ধরুন, আপনি একটি Transaction COMMIT করলেন। `LGWR` ইতিমধ্যে Redo Information Online Redo Log File-এ Write করেছে, কিন্তু `DBWn` এখনো সেই পরিবর্তিত Block Datafile-এ Write করেনি। এর মধ্যেই হঠাৎ Database Crash করল।

তাহলে কি আপনার Committed Data হারিয়ে যাবে?

**না।** Database আবার চালু হওয়ার সময় Oracle **Instance Recovery**-এর মাধ্যমে Redo Information ব্যবহার করে Committed Changes Recover করতে পারে। মানে, ডায়েরিতে লেখা আছে বলেই কাজটা ফিরিয়ে আনা সম্ভব — প্লেট ধোয়া তখনো বাকি থাকলেও।

তবে একটা কথা মনে রাখবেন — এটা **Instance/Crash Recovery**, **Media Recovery নয়**। যদি Redo Log File নিজেই নষ্ট হয়ে যায় (যেমন disk failure), তাহলে গল্পটা ভিন্ন — তখন `ARCHIVELOG` mode আর RMAN backup দরকার। এই দুটো গুলিয়ে ফেলা interview-এ খুব common ভুল।

## তাহলে COMMIT কি কখনো wait করে?

হ্যাঁ করে — কিন্তু Datafile-এর জন্য নয়, **Redo Flush**-এর জন্য।

`COMMIT WRITE IMMEDIATELY` (default) হলে COMMIT synchronously `LGWR`-এর redo write শেষ হওয়ার জন্য অপেক্ষা করে। এই অপেক্ষার নামই **`log file sync`**। আপনি `V$SESSION_EVENT`-এ এটা দেখতে পাবেন। যদি `log file sync` বেশি হয়, বুঝবেন আপনার redo disk-এ I/O bottleneck আছে।

## মনে রাখার সারাংশ

| Process | কখন লেখে | কোথায় | কেন |
| :--- | :--- | :--- | :--- |
| **LGWR** | COMMIT-এ (এবং log buffer full / 3 sec / 1/3 full) | Online Redo Log | Durability — WAL contract |
| **DBWn** | Checkpoint target অনুযায়ী batch-এ | Datafile | Random I/O কমানো, recovery bounded রাখা |
| **CKPT** | Checkpoint-এ | Control file + datafile header | DBWn-এর progress track করা |

তাই একজন DBA হিসেবে আমাদের মনে রাখতে হবে, **COMMIT সফল হওয়া মানেই Datafile-এ Data Write হয়ে যাওয়া নয়। COMMIT সফল মানে — Redo Disk-এ Safe।**

Oracle Database-এর ভেতরে কীভাবে কাজগুলো হয়, তা বোঝার জন্য শুধু SQL Command জানলেই হবে না। এর পেছনের Architecture এবং Background Process-গুলোর কাজও বোঝা জরুরি।

কারণ Interview Board-এ অনেক সময় প্রশ্নটা কঠিন হয় না, কঠিন হয় এর পেছনের Logic ব্যাখ্যা করা।
