# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** ยสินทร สันนิภางกูร

**Student ID / รหัสนิสิต:** 6630275021

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 499 | 589 | 93.5% | 100% |
| t0-small | 11 | 35 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

>ให้ accuracy ที่มากกว่า แต่แลกกับ p50ms และ p95ms ที่มากกว่า เช่นเดียวกับ largecall 100% แปลว่าทุกรีวิวต้องจ่ายเงิน และ capacity จำกัดที่ 20calls/s

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 25.9 | 258 | 436 | 18 | 0 | 92.9% |
| t1-after | 9.2 | 397 | 1007 | 0 | 25 | 93.7% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

>เพราะ requests ที่ error จะไม่ถูกนำมาคิด latency. column errors

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

>requests ที่ค้างถือ connection, coroutine และ memory ใน review service และ gateway นานถึง 20s แทนที่จะเป็น 0.2s ทรัพยากรมีจำกัดการค้างไม่กี่ตัวจึงกิน capacity ส่วนใหญ๋ ส่งผลให้ client หมดเวลาก่อน รีวิวจึงหายไป

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

>เก็บรีวิว pending ไว้ แล้วให้ batch job หรือ worker มาดึงไปติด label ทีหลังเมื่อ large model กลับมาปกติ topology ที่ทำแบบนี้คือ T4

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

>large model ใช้เวลาประมาณ 200 ms ต่อ call และยังมีเวลารอคิว 0.1 วินาทีจึงสั้นกว่าเวลาปกติ เกือบทุกรีวิวจะกลายเป็น pending แม้ระบบไม่ได้มีปัญหาอะไร ผู้ใช้จึงไม่ได้ label ทันที ผิด FR2 และจ่ายค่า large call ไปเปล่าๆด้วย เพราะ call ยังรันจนเสร็จฝั่ง inference ส่วนตั้ง 10s นานเกินไป จะผิด QR2 และ request ที่ค้างยังถือทรัพยากรไว้นาน 10 วินาที ระบบจึงยังช้าทั้งระบบเหมือนเดิม

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 5 | 546 | 91.5% | 38% |
| t2-0.7 | 0.7 | 362 | 577 | 93% | 53% |
| t2-0.8 | 0.8 | 438 | 570 | 93.5% | 64% |
| t2-0.9 | 0.9 | 479 | 590 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

>0.7 เพราะมี accuracy ไม่ต่างจาก threshold 0.8-0.9 large call น้อยกว่า แต่แลกกับ p50ms และ p95ms ที่สูงกว่า 0.6 threshold ไม่เลือก 0.6 เพราะเฉียดเส้นเกินไปและ quality ที่ยอมเสียคือ cost เมื่อเทียบกับ 0.6

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

>เพราะรีวิวที่ง่ายที่ small ตอบได้ดึง p50 ลงแต่ p95 คือรีวิวยากที่ต้องรอ large model เหมือนเดิม

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

>small ทั่นใจแต่ผิดกับประโยคปฏิเสธหรือประชด ทำให้รีวิวไม่ถูกส่งต่อไป large เจ้าของร้านจึงไม่เห็นลูกค้าที่ไม่พอใจ

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 4 | 545 | 93% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

>รีวิวที่มีข้อความเหมือนกันจะได้ประโยชน์จาก cache มากที่สุด และรีวิวที่ไม่เหมือนใครเลยจะไม่ได้ประโยชน์จากแคช

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

>ต้องล้าง cache เก่า และออกแบบโดยใส่ model version ใน cache key

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

>t1-before p50 ดูดีกว่า baseline เพราะ request ที่ error ไม่ถูกนับใน latency ตัวเลข percentile จึงหลอกได้ถ้าไม่ดู errors ควบคู่
