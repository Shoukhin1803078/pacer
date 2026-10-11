# pacer
তোমার PACER (Prediction-Aware Context-Efficient Retrieval) architecture-এর মূল উদ্দেশ্য হলো CPU-only environment-এ RAG-এর latency কমানো, কিন্তু answer quality যেন বেশি কমে না যায়।

সহজভাবে বললে, PACER একসঙ্গে তিনটি বিষয় নিয়ন্ত্রণ করে:

1. কখন retrieval করতে হবে — Prediction
2. কতটুকু retrieved evidence ব্যবহার করতে হবে — Compression/Budgeting
3. কখন আগের cache পুনরায় ব্যবহার করা যাবে — Cache reuse, যা Phase 2-তে optional

সবচেয়ে গুরুত্বপূর্ণ বিষয় হলো, PACER প্রতিটি সিদ্ধান্তের লাভ ও খরচ হিসাব করে।

## ১. Setup — Experiment-এর প্রস্তুতি

Data

HotpotQA হলো primary dataset। পাশাপাশি 2Wiki, Natural Questions (NQ) এবং TriviaQA ব্যবহার করার পরিকল্পনা রয়েছে।

Machine and Model

একই CPU machine-এ একটি ছোট open-source LLM, যেমন 3B parameter model, চালানো হবে। GPU ব্যবহার করা হবে না।

Retriever

BM25 ব্যবহার করা হবে। Dense retrieval index optional।

Disjoint data splits

Predictor-এর calibration data এবং test data আলাদা থাকবে। Test data untouched থাকবে, যাতে evaluation-এর ফল test data দিয়ে tuning করার কারণে biased না হয়।

## ২. PACER Controller — Architecture-এর মূল অংশ
<img width="3999" height="2250" alt="PACER_methodology_CPU" src="https://github.com/user-attachments/assets/b509ff4e-e9fd-477c-a266-294af4bc696f" />

এখানে controller একটি নতুন retriever বা নতুন LLM নয়। এটি বিদ্যমান components-এর ওপর বসে সিদ্ধান্ত নেয়, কোন কাজটি কখন এবং কতটুকু করা উচিত।

User Query

ব্যবহারকারীর প্রশ্ন

M1 — Retrieval-Need Predictor

প্রশ্নের বৈশিষ্ট্য এবং local model-এর signal ব্যবহার করে অনুমান করে retrieval দরকার কি না। এটি retrieval-এর সম্ভাবনা \\(p\\) এবং সিদ্ধান্ত নেওয়ার probe cost সম্পর্কে তথ্য দেয়।

M3 — Net-Benefit Gate

Retrieval বা অন্য কোনো action থেকে সম্ভাব্য latency saving, সেই action-এর মোট খরচের চেয়ে বেশি কি না যাচাই করে। লাভ যথেষ্ট না হলে standard RAG ব্যবহার করবে।

M5 — Lookahead Retrieval

Retrieval উপকারী এবং gate-এর শর্ত পূরণ হলে, generation শেষ হওয়ার অপেক্ষা না করে আগেই retrieval শুরু করতে পারে।

M2 — Adaptive Evidence Budget

Retrieved evidence-এর কোন অংশ রাখা হবে, কোন অংশ বাদ যাবে এবং কখন compression বন্ধ হবে, তা নির্ধারণ করে।

M6 — Local LLM

CPU-তে চলা LLM প্রয়োজনীয় evidence ব্যবহার করে final answer তৈরি করে।

M7 — Telemetry

প্রতিটি stage-এর সময় ও resource cost log করে। এই measured cost আবার Gate-এ ফেরত যায়, যাতে পরবর্তী সিদ্ধান্ত আরও বাস্তবসম্মত হয়।

M4 — KV Cache (Phase 2, optional)

আগের prefilled state পুনরায় ব্যবহার করা যায় কি না পরীক্ষা করবে। এটি মূল PACER experiment-এর আবশ্যিক অংশ নয়।

### প্রতিটি module কী কাজ করে?

| Module       | সহজ ব্যাখ্যা                                      |
| ------------ | ------------------------------------------------- |
| M1 Predictor | Retrieval দরকার হওয়ার সম্ভাবনা অনুমান করে।       |
| M2 Budget    | কতটুকু evidence রাখা উচিত তা নির্ধারণ করে।        |
| M3 Gate      | সম্ভাব্য লাভ বনাম মোট খরচ যাচাই করে।              |
| M4 Cache     | আগের computation পুনরায় ব্যবহার করে; optional।   |
| M5 Retrieval | প্রয়োজন হলে আগেভাগে document retrieval শুরু করে। |
| M6 Local LLM | Retrieved evidence দিয়ে উত্তর তৈরি করে।          |
| M7 Telemetry | বাস্তব সময় ও resource cost মেপে Gate-কে জানায়।  |

## ৩. PACER-এর দুটি গুরুত্বপূর্ণ decision rule

M2 — Evidence Budget Rule

তোমার architecture অনুযায়ী, কোনো sentence রাখা হবে যদি তার score সেরা score-এর তুলনায় নির্ধারিত threshold পূরণ করে। পাশাপাশি মোট retained token-এর সংখ্যা \\(B\_{\max}\\)-এর মধ্যে থাকতে হবে।

\\[ s_i \geq p \times s\_{\max} \\]

এখানে:

- \\(s_i\\) = evidence sentence-এর score
- \\(s\_{\max}\\) = সর্বোচ্চ score
- \\(p\\) = Predictor-এর retrieval-need probability
- \\(B\_{\max}\\) = সর্বোচ্চ token budget

অর্থাৎ, শুধু relevant evidence রাখার চেষ্টা করবে এবং অপ্রয়োজনীয় prompt ছোট করবে।

M3 — Net-Benefit Gate Rule

\\[ \text{Expected Saving} - \text{Total Cost} > \tau \\]

এখানে \\(\tau\\) হলো একটি minimum benefit threshold। Architecture-এ retrieval skip করার আগে \\(p \geq c\_{\min}\\) শর্তও আছে।

Total cost-এর মধ্যে probe, retrieval, compression এবং প্রাসঙ্গিক transfer overhead অন্তর্ভুক্ত করতে হবে। ফলে predictor ব্যবহার করতে গিয়েই যদি অতিরিক্ত সময় নষ্ট হয়, PACER যেন ভুলভাবে নিজেকে দ্রুত দাবি না করে।

## ৪. Ablation Study — কীভাবে প্রমাণ করবে PACER কাজ করে?

এটি তোমার methodology-র অন্যতম গুরুত্বপূর্ণ অংশ। সব system একই CPU, model এবং query ব্যবহার করবে, যাতে comparison fair হয়।

| System | Prediction      | Compression     | Cache | উদ্দেশ্য                  |
| ------ | --------------- | --------------- | ----- | ------------------------- |
| S0     | ✗               | ✗               | ✗     | Standard RAG baseline     |
| S1     | ✓               | ✗               | ✗     | Prediction-এর প্রভাব      |
| S2     | ✗               | ✓               | ✗     | Compression-এর প্রভাব     |
| S3     | ✗               | ✗               | ✓     | Cache-এর প্রভাব           |
| S4     | ✓               | ✓               | ✗     | PACER core — primary test |
| S5–S7  | বিভিন্ন সমন্বয় | বিভিন্ন সমন্বয় | ✓     | Optional cache extension  |

S4 হলো মূল পরীক্ষা। এটি S0, S1 এবং S2-এর তুলনায় latency কমাতে পারে কি না এবং answer quality বজায় রাখতে পারে কি না, সেটিই primary research question।

## ৫. Evaluation — কী কী measure করবে?

Latency

- TTFT: প্রথম token পেতে কত সময় লাগে।
- End-to-end latency: পুরো উত্তর পেতে কত সময় লাগে।
- Mean, P50 ও P95 latency।
- Predictor/probe-এর সময়সহ প্রতিটি stage-এর cost।

Answer Quality

- Exact Match (EM)
- Token F1
- Evidence recall

লক্ষ্য হলো latency কমানো, কিন্তু answer quality গ্রহণযোগ্য সীমার বাইরে না নামানো।

Resource Usage

- Peak RAM
- Retrieval calls
- অপ্রয়োজনীয় retrieval
- Compression ও controller overhead

Statistical Analysis

Paired bootstrap, quality non-inferiority test (TOST), Holm correction এবং interaction analysis ব্যবহার করে ফলাফলের নির্ভরযোগ্যতা যাচাই করা হবে।

## ৬. শেষ পর্যন্ত কী ফলাফল হতে পারে?

তোমার architecture তিনটি ফলাফলকেই বৈজ্ঞানিকভাবে গ্রহণযোগ্য বলে ধরে।

A — Gain

PACER-এর latency কমে, quality নির্ধারিত margin-এর মধ্যে থাকে এবং coordination-এর সুবিধা প্রমাণিত হয়। তখন PACER ব্যবহার করার পক্ষে evidence থাকবে।

B — Match

PACER-এর ফল standard বা single-technique RAG-এর কাছাকাছি। তখন সহজ system-টি ব্যবহার করাই যুক্তিযুক্ত হতে পারে।

C — Loss

Controller, prediction বা compression-এর overhead সম্ভাব্য লাভের চেয়ে বেশি হয়। তখন coordinated approach-এর সীমাবদ্ধতা এবং কোন workload-এ এটি ব্যর্থ হয়, তা বিশ্লেষণ করতে হবে।

তোমার প্রস্তাবিত success criterion অনুযায়ী, S4-এর probe-inclusive mean এবং P95 latency—S0, S1 ও S2-এর তুলনায় অন্তত ১০% কম হতে হবে; quality loss নির্ধারিত margin-এর মধ্যে থাকতে হবে এবং interaction analysis-ও ইতিবাচক হতে হবে।

## পুরো architecture এক বাক্যে

PACER প্রথমে অনুমান করে retrieval দরকার কি না, তারপর সম্ভাব্য লাভ ও খরচ বিচার করে retrieval timing এবং evidence budget সমন্বয় করে, CPU-তে ছোট LLM দিয়ে উত্তর তৈরি করে এবং প্রতিটি stage-এর বাস্তব cost মেপে নিজের সিদ্ধান্ত যাচাই করে।

এটাই তোমার গবেষণার মূল দাবি: CPU-only RAG-এ আলাদা আলাদা optimization চালানোর পরিবর্তে shared latency budget-এর অধীনে তাদের সমন্বয় করলে কি কম latency-তে একই মানের উত্তর দেওয়া সম্ভব?
