# Corvin v3 — pitch script (≈ 3 minutes) + Q&A prep

Open `v3/` with **Presenter view** on (the default). Set the browser to full screen.

## 1 · Hook (20 s)

> "Every year a new GPU generation comes out, and each one needs more power. More power means more heat, and more heat means more parts failing: PSUs and fans first. Today a technician walks to the rack, opens a ticket, swaps the part and checks it. That takes about 38 minutes of hands-on time and 2½ hours from alarm to a working server. That work is repetitive and high-volume, the kind the chairman said AI should take over."

## 2 · Live demo (60 s): press **▶ Start demo**

The view follows the gripper while you talk:

- "The servers report which part failed. Corvin reads the **ArUco marker** to find the exact slot, and the **barcode** to prove it's the right server. It never swaps the wrong machine."
- "It pulls the failed part, drops it in the **RMA bin**, takes a spare from the caddy and inserts it. Then the status LED confirms the fix."
- Point at **Corvin AI · live** on the right: "Every swap is analysed right away: how long the part was broken, whether this part type keeps failing, and when to restock."

The dashboard opens by itself when the rack is healthy again.

## 3 · Dashboard (60 s): the value is in the decisions

- **Weekly briefing, card 1:** "The top failing part isn't just a restock signal, it's a buying decision. Corvin checks two things:
  - **load and heat**: do the failures follow rack power draw (here r = 0.8)? If yes, the part runs too close to its rating, so buy a *higher-rated series*.
  - **supplier quality**: is the failure rate far above the benchmark (here 3.7×)? If yes, the brand is the problem, so *change the brand*.

  For PSUs it's both. So the action is to *upgrade the series, and A/B-test a second supplier on 2 racks before switching*."
- **Recent breakdowns:** "One part type, PSUs from Supplier D, causes about a quarter of all downtime. That's where a better part pays back fastest."
- **Small cards:** the technician-hours saved, the spare-parts order with lead time, and the effect of BMC fault codes on time broken.
- **Condition grid:** "Worn parts are swapped *before* they fail, in the lowest-load window."

## 4 · Close (20 s)

> "Corvin takes the repetitive swap off the technician and returns hours plus better decisions. People keep the work AI can't do: design, new technology, and the judgement calls. It's a small robot today, and the same software scales to every rack."

## Likely questions

| Question | Answer |
|---|---|
| *What if the robot fails mid-swap?* | Turn off Presenter view, switch on **Simulate automation failure** and run Auto-Swap. Corvin stops, holds position and hands over to manual control (keyboard or gamepad). It never guesses near live hardware. E-STOP is always on screen. |
| *Is the AI real?* | The insights are computed from the data on the page: failure rate vs a benchmark, concentration by position and time, correlation with rack load, inventory lead time. That's deliberately explainable: every claim shows its evidence. The next step is training a model on real BMC and maintenance logs. |
| *Why not a mobile robot?* | v2 had one. For budget and prototype scope, v3 is stationary: one gantry per rack row is simpler, safer and cheaper. The software (planning, ArUco + barcode, the dashboard) carries over. |
| *Where do the numbers come from?* | The weekly history is mock data for a 9,600-server site. Rack R-01 is live from the simulation. The assumptions (38 min hands-on, 2.5 h manual MTTR, benchmark AFR) are listed in the dashboard footnote. Say this before a judge asks. |
| *Power trend?* | GPU power keeps rising each generation, and PSU/fan stress rises with it. That's why the PSU insight says "higher-rated series". It matches the move to higher-power racks and 800 V DC. |
| *Battery / energy?* | Same pattern as the BMS discussion: per-cell sensing, a large dataset, then AI to spot outliers. Corvin's condition grid is that idea applied to server parts. |
