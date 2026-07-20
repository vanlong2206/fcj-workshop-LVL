---
title: "Verification and Testing"
date: 2026-07-13
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---
## 5.10.1 Demo gameplay after completing the Game Backend storage and processing system

<div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
  <iframe src="https://www.youtube.com/embed/bxgfVyOcEXw" 
          style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border:0;" 
          allowfullscreen 
          title="Kiểm tra kết quả và thực nghiệm">
  </iframe>
</div>

## 5.10.2 Check SQS DLQ and Maintenance for 2D game

<div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
  <iframe src="https://www.youtube.com/embed/Vmwd8U29pL0" 
          style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border:0;" 
          allowfullscreen 
          title="Verification and Testing">
  </iframe>
</div>

##### * Context:

Test with **50 virtual users** running in parallel for **5 minutes**, each user performs: Register → Login → [ Earn(100 coin) → Spend(random) ] loop

**30% spend** uses amount=999.999.999 — intentionally larger than the balance to cause failure leading to DLQ
