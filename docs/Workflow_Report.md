# PathSnap: Distributed System Architecture & Evaluator Workflow Report

**Project Title:** PathSnap — Distributed Map-Matching & Route Reconstruction  
**Evaluation Scope:** Two-Laptop Client-Server Demonstration & Technical Defense  
**Author / Team:** Harshitha B. T. & Team (`Harshith493/PathSnap`)  
**Target Environment:** Local Area Network (Wi-Fi / Ethernet Hotspot)  

---

## 1. Executive Summary & Defense Keypoints

This report describes the operational workflow, network connection protocol, algorithmic lifecycle, and theoretical defense points for the **PathSnap Two-Laptop Distributed System**.

```
[ Laptop 2: Evaluator UI ]                            [ Laptop 1: API Server & ML Engine ]
   (Untrusted Client)                                        (Trusted Backend Core)
┌─────────────────────────┐                                 ┌────────────────────────────────────┐
│ • Browser (Chrome/Edge) │                                 │ • FastAPI + Uvicorn (:8000)        │
│ • Dropdown (10 Trips)   │  1. GET /trips                  │ • Model: HistGradientBoosting      │
│ • Metric Scorecards     │ ──────────────────────────────► │ • Bayesian Tuned (Optuna TPE)      │
│ • Folium Map Viewport   │ ◄────────────────────────────── │ • HMM Viterbi Sequential Decoder   │
│                         │     JSON: [10 Trip IDs]         │ • Ground Truth: evaluator_test/    │
│                         │                                 │   (Strictly Server-Side)           │
│                         │  2. POST /predict/00000049      │                                    │
│ [Run Evaluation Button] │ ──────────────────────────────► │ [Candidate Gen] -> [12-Features]   │
│                         │                                 │ [ML Predict]    -> [HMM Viterbi]   │
│ • Top-1 Accuracy: 94.8% │                                 │ [Score Against Ground Truth]       │
│ • Continuity:     99.7% │ ◄────────────────────────────── │                                    │
│ • Gain:          +3.3%  │     JSON: Evaluation Metrics    │                                    │
│                         │           + Route Coordinates   │                                    │
│                         │                                 │                                    │
│                         │  3. GET /map/00000049           │                                    │
│ • Layer 1: GPS Trace    │ ──────────────────────────────► │                                    │
│ • Layer 2: Clf Route    │ ◄────────────────────────────── │                                    │
│ • Layer 3: HMM Route    │     HTML: Interactive Map       │                                    │
└─────────────────────────┘                                 └────────────────────────────────────┘
```

---

## 2. Two-Laptop Network Connection & Setup Guide

### 2.1 Network Topology & Prerequisites
Both laptops must be connected to the **same local subnet**. You do not need active internet access once dependencies are installed. A phone hotspot, Wi-Fi router, or direct Ethernet cable works.

### 2.2 Step-by-Step Connection Instructions

#### Phase A: Laptop 1 Configuration (Server / API Engine)
1. **Connect to Wi-Fi / Hotspot:** Connect Laptop 1 to the network.
2. **Retrieve Local IPv4 Address:**
   Open PowerShell and run:
   ```powershell
   ipconfig
   ```
   Locate your active adapter (e.g., `Wireless LAN adapter Wi-Fi`) and note the **IPv4 Address** (e.g., `192.168.1.45` or `192.168.137.1`).
3. **Launch the FastAPI Backend:**
   In PowerShell at the project directory:
   ```powershell
   cd "c:\Users\Harshitha.B.T\OneDrive\Desktop\5th_Semester_2026\ML\Hackathon"
   python src/server.py
   ```
   *Expected startup log:*
   ```text
   [INFO] [PathSnap-API] Loading trained model artifact from: artifacts/models/gradient_boosting_tuned.joblib
   [INFO] [PathSnap-API] Model and HMM Viterbi decoder loaded successfully!
   INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
   ```
4. **Firewall Verification (If Connection Fails):**
   If Laptop 2 cannot connect, ensure port 8000 is open in Windows Defender Firewall:
   ```powershell
   New-NetFirewallRule -DisplayName "PathSnap API" -Direction Inbound -LocalPort 8000 -Protocol TCP -Action Allow
   ```

#### Phase B: Laptop 2 Configuration (Evaluator UI Dashboard)
Laptop 2 is the client where the professor / evaluator operates.
1. **Connect to the Same Wi-Fi / Hotspot** as Laptop 1.
2. **Access the Evaluator Dashboard:**
   Open Google Chrome or Microsoft Edge on Laptop 2 and navigate to:
   ```text
   http://<LAPTOP_1_IP>:8000/ui
   ```
   *(Example: `http://192.168.1.45:8000/ui`)*
   
   *(Alternative)* If Laptop 2 has the code cloned locally:
   ```powershell
   python run_ui.py
   ```
   Then open `http://localhost:8080` and enter `http://<LAPTOP_1_IP>:8000` in the "Server Endpoint" bar.

3. **Verify Connectivity:**
   The UI will immediately turn green showing: `Server Status: Connected` and populate the dropdown with the **10 Reserved Trips**.

---

## 3. The 10 Reserved Evaluator Trips & Ground-Truth Security

### 3.1 Trip Directory Structure
The 10 reserved trips reside **exclusively on Laptop 1** in:
`evaluator_test/evaluator_test/`

| Reserved Trip ID | Raw GPS Observations | Road Network Arcs | Trip Duration | Path Characteristics |
|:---:|:---:|:---:|:---:|:---|
| **00000015** | 6,854 pts | 612 arcs | ~114 min | Long suburban arterial with highway merge |
| **00000017** | 1,346 pts | 218 arcs | ~22 min | Dense downtown grid with tall building shadows |
| **00000023** | 458 pts | 94 arcs | ~7 min | Short local corridor; high sampling stability |
| **00000030** | 654 pts | 142 arcs | ~11 min | Multi-lane junction with complex turn maneuvers |
| **00000049** | 2,642 pts | 341 arcs | ~44 min | Mixed highway and urban ring road *(Best Demo Trip)* |
| **00000059** | 1,031 pts | 186 arcs | ~17 min | Secondary residential roads with noisy GPS drift |
| **00000067** | 916 pts | 175 arcs | ~15 min | Overpass / underpass non-planar geometry |
| **00000073** | 1,144 pts | 204 arcs | ~19 min | Linear rural-to-urban transit segment |
| **00000087** | 1,399 pts | 258 arcs | ~23 min | High-curvature roundabouts and dual carriageways |
| **00000094** | 3,230 pts | 487 arcs | ~53 min | Complex industrial zone with missing satellite lock |

### 3.2 Defense Answer: Why is Ground Truth Kept Server-Side?
> **Viva Question:** *"Why doesn't Laptop 2 load the `.route` file to compute accuracy itself?"*  
> **Answer:**  
> **Zero-Leakage Architectural Integrity.** In real-world telematics and competitive hackathons, the client is an untrusted endpoint. If ground-truth labels (`.route`) were transmitted to the client, the client could trivially overfit, cache answers, or alter metric computation. Laptop 1 acts as the authoritative verification oracle: it accepts only raw trip identifiers, executes feature engineering and inference in a private sandbox, scores the predictions against held-out server reference data, and returns signed evaluation metrics.

---

## 4. End-to-End Inference & Algorithmic Pipeline

When the evaluator selects **Trip `00000049`** and clicks **"Run Evaluation"**, the following 7-stage pipeline executes on Laptop 1:

```
[Raw GPS Track] ──► [Equirectangular Projection] ──► [Dual KDTree Candidate Gen]
                                                              │
                                                              ▼
[Route Continuity Metrics] ◄── [Viterbi DP] ◄── [HistGBT Prob] ◄── [12-Feature Extraction]
```

### Stage 1: Spatial Coordinate Projection
Raw GPS points are provided in WGS84 Geodetic coordinates (longitude $\lambda$, latitude $\phi$ in degrees). Euclidean distance calculations in degrees cause severe latitude distortion ($\approx 111\text{ km/deg}$ latitude vs $\approx 71\text{ km/deg}$ longitude at $49^\circ\text{N}$).  
We apply local **Equirectangular Projection** centred at the trajectory centroid $(\lambda_0, \phi_0)$:
$$x = R \cdot (\lambda - \lambda_0) \cdot \cos\left(\frac{\phi_0 \cdot \pi}{180}\right) \cdot \frac{\pi}{180}, \quad y = R \cdot (\phi - \phi_0) \cdot \frac{\pi}{180}$$
where $R = 6,371,000\text{ m}$. All subsequent geometric calculations operate in orthogonal metric metres.

### Stage 2: Dual KDTree Candidate Generation
For each GPS point $p_i$, exhaustive distance computation against thousands of road network arcs is $O(N \cdot M)$. We build two spatial KDTrees:
1. **Arc Midpoint KDTree:** Queries $k=15$ nearest road segment midpoints.
2. **Intersection Node KDTree:** Queries $k=15$ nearest network vertices and retrieves connected incident arcs.
The union yields top-$k$ ($k=10$) unique candidate arcs per observation point.
*Measured Performance:* **95.97% recall** (ground-truth arc is within top-10 candidates across 250,000+ points).

### Stage 3: 12-Dimensional Feature Engineering
For every (GPS point $p_i$, Candidate Arc $a_j$) pair, we extract 12 scale-invariant geometric and kinematic features:
1. `perp_dist`: Orthogonal Euclidean distance from GPS point to arc line segment (metres).
2. `proj_t`: Scalar projection factor along segment $t \in [0, 1]$ ($0 = \text{start node } u$, $1 = \text{end node } v$).
3. `dist_to_u`: Distance from GPS point to arc start node $u$ (metres).
4. `dist_to_v`: Distance from GPS point to arc end node $v$ (metres).
5. `arc_length`: Physical length of the candidate road segment (metres).
6. `heading_diff`: Angular error $|\theta_{\text{GPS}} - \theta_{\text{arc}}|$ bounded in $[0^\circ, 180^\circ]$.
7. `dist_rank`: Integer proximity rank ($1 = \text{closest spatial candidate}$).
8. `delta_dist`: Proximity margin: $\text{perp\_dist}(a_j) - \text{perp\_dist}(a_{\text{closest}})$.
9. `speed`: Instantaneous GPS velocity estimate ($\Delta d / \Delta t$ in m/s).
10. `speed_length_ratio`: Ratio of velocity to arc length (traversal feasibility).
11. `u_degree`: Topological out-degree of start intersection $u$.
12. `v_degree`: Topological in-degree of end intersection $v$.

### Stage 4: Machine Learning Classification
- **Model:** Histogram-Based Gradient Boosted Decision Trees (`HistGradientBoostingClassifier`).
- **Input:** $X \in \mathbb{R}^{K \times 12}$ where $K = \sum k_i$ (total candidate pairs for the trip).
- **Output:** Posterior probability $P(Y = 1 \mid \mathbf{x}) \in [0, 1]$ that candidate arc $a_j$ is the true traversing road segment.
- **Inference Speed:** ~180 ms for 26,420 candidates on CPU.

### Stage 5: Sequential HMM Viterbi Decoding
Independent classifier predictions suffer from **disjoint zigzagging** (selecting high-probability arcs that do not physically connect). We formulate route reconstruction as a Hidden Markov Model:
- **Hidden States:** True road arc $s_t \in \{a_1, \dots, a_k\}$ at timestep $t$.
- **Observations:** GPS measurement sequence $o_1, o_2, \dots, o_T$.
- **Emission Probability:** Combines ML discriminative probability with Gaussian spatial decay:
  $$P(o_t \mid s_t) = \alpha \cdot P_{\text{ML}}(s_t \mid o_t) + (1 - \alpha) \cdot \frac{1}{\sqrt{2\pi}\sigma} \exp\left(-\frac{d(o_t, s_t)^2}{2\sigma^2}\right)$$
  *(Parameters: $\alpha = 0.75$, $\sigma = 12.0\text{ m}$ GPS error standard deviation).*
- **Transition Probability:** Topology-driven road network connectivity matrix:
  $$P(s_t \mid s_{t-1}) = \begin{cases} 
  0.65 & \text{if } s_t = s_{t-1} \text{ (vehicle continues on same arc)} \\
  0.90 & \text{if } \text{target}(s_{t-1}) = \text{source}(s_t) \text{ (1-hop valid network connection)} \\
  0.50 & \text{if reachable within 2 network hops} \\
  10^{-5} & \text{if disconnected (severe penalty for impossible teleports)}
  \end{cases}$$
- **Viterbi Dynamic Programming (Log-Space):**
  $$\delta_t(s) = \max_{s' \in \text{Cand}_{t-1}} \left[ \delta_{t-1}(s') + \ln P(s \mid s') \right] + \ln P(o_t \mid s)$$
  Backtracking from $t = T$ recovers the globally optimal, topologically continuous road sequence.

### Stage 6: Real-Time Evaluation Against Server Ground Truth
Laptop 1 aligns the ground truth `.route` arcs to observations using closest-projection indexing and calculates:
1. **Top-1 Pointwise Accuracy:** $\frac{1}{T} \sum_{t=1}^T \mathbb{I}(\hat{s}_t == s_t^*)$
2. **Route Continuity:** Fraction of consecutive arc transitions that share a common node in the network:
   $$\text{Continuity} = \frac{1}{T-1} \sum_{t=1}^{T-1} \mathbb{I}\left( \text{target}(\hat{s}_t) == \text{source}(\hat{s}_{t+1}) \lor \hat{s}_t == \hat{s}_{t+1} \right)$$
3. **Candidate Miss Rate:** Percentage of timesteps where the true arc was absent from the top-10 candidate pool.

---

## 5. Official Evaluator Test Set Benchmark Results

All metrics below are derived from actual execution on the 10 held-out `evaluator_test` trips:

| Trip ID | GPS Points | Classifier Top-1 Acc | Classifier Continuity | HMM Viterbi Top-1 Acc | HMM Viterbi Continuity | Continuity Gain | Total Server Latency |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **00000015** | 6,854 | 52.22% | 90.98% | 49.07% | **100.00%** | **+9.02%** | 11.0 s |
| **00000017** | 1,346 | 87.44% | 80.59% | 76.89% | **97.70%** | **+17.10%** | 3.1 s |
| **00000023** | 458 | 90.83% | 93.65% | 89.30% | **97.59%** | **+3.94%** | 0.8 s |
| **00000030** | 654 | 74.92% | 76.72% | 71.10% | **97.55%** | **+20.83%** | 1.3 s |
| **00000049** | 2,642 | 96.18% | 96.40% | 94.81% | **99.73%** | **+3.33%** | 4.2 s |
| **00000059** | 1,031 | 90.11% | 85.53% | 82.15% | **98.45%** | **+12.91%** | 1.7 s |
| **00000067** | 916 | 65.39% | 94.10% | 59.83% | **100.00%** | **+5.90%** | 1.3 s |
| **00000073** | 1,144 | 91.43% | 93.26% | 89.16% | **100.00%** | **+6.74%** | 1.9 s |
| **00000087** | 1,399 | 92.85% | 91.42% | 97.28% | **100.00%** | **+8.58%** | 4.5 s |
| **00000094** | 3,230 | 71.70% | 87.40% | 67.86% | **99.94%** | **+12.54%** | 4.6 s |
| **OVERALL AVG** | **1,967** | **81.31%** | **89.00%** | **77.75%** | **99.10%** | **+10.09%** | **3.5 s** |

### Key Analytical Takeaways:
- **Zero Disjoint Breakages:** HMM Viterbi achieves **$\ge 97.5\%$ continuity on every single trip**, with 4 trips achieving **100.00% perfect topological continuity**.
- **The Pointwise Accuracy vs. Continuity Trade-off:** The HMM trades a minor $\approx 3.5\%$ pointwise classification accuracy to prevent physically impossible road leaps. When a GPS reading drifts near an intersection, the classifier might choose an adjacent one-way street; Viterbi overrides this with the continuous network path. For map-matching, topological continuity is the defining objective.

---

## 6. Comprehensive Defense Q&A for Technical Evaluation

### Q1: "Which machine learning model did you choose and why?"
> **Answer:**  
> We benchmarked 5 distinct model families under 3-fold group-stratified cross-validation: Logistic Regression, Distance-Weighted KNN ($k=15$), Decision Trees, Random Forest, and **Histogram Gradient Boosted Trees (`HistGradientBoostingClassifier`)**.  
> We selected **HistGradientBoosting** because:
> 1. **Highest Cross-Validation F1-score:** Achieved **0.8857 CV Macro F1** after Optuna Bayesian tuning.
> 2. **Superb Probability Calibration (Log-Loss = 0.0919):** Calibration is paramount because output probabilities are directly injected into the HMM emission density. Models that produce overconfident 0/1 probabilities (like single Decision Trees) starve the Viterbi search.
> 3. **Efficiency:** Processes ~26,000 candidate feature vectors in under 200 ms.

### Q2: "What is Bayesian Optimization doing in your project?"
> **Answer:**  
> Rather than arbitrary manual tuning or exhaustive grid search, we used **Optuna with Tree-structured Parzen Estimator (TPE)** over 20 iterations to maximize 3-fold grouped CV Macro F1.  
> It searched across:
> - `learning_rate` $\in [0.01, 0.3]$ (Best: **0.0304**)
> - `max_iter` $\in [50, 300]$ (Best: **207**)
> - `max_depth` $\in [3, 12]$ (Best: **8**)
> - `min_samples_leaf` $\in [5, 50]$ (Best: **26**)
> - `l2_regularization` $\in [10^{-5}, 10]$ (Best: **0.00136**)  
> Parameter importance analysis identified `learning_rate` (51% relative importance) and `min_samples_leaf` (26% relative importance) as the two primary drivers of generalization.

### Q3: "What API technology did you build and how are endpoints structured?"
> **Answer:**  
> We implemented an asynchronous **FastAPI** REST backend deployed via **Uvicorn** with full CORS middleware.
> 1. `GET /health`: Verifies server status, loaded `.joblib` model architecture, and active evaluation directory.
> 2. `GET /trips`: Dynamically introspects the held-out dataset and returns all 10 available trip IDs to populate client dropdowns.
> 3. `POST /predict/{trip_id}`: Core inference engine. Loads trip data, generates candidates, extracts features, runs HistGBT, executes Viterbi decoding, computes ground-truth accuracy, and returns JSON performance scorecards + coordinate trajectories.
> 4. `GET /map/{trip_id}`: Renders an interactive Folium Leaflet map with toggleable visual layers.
> 5. `GET /ui`: Serves the responsive Bootstrap 5 evaluator web application directly.

### Q4: "How does your system prevent GPS noise and multipath errors?"
> **Answer:**  
> GPS multipath errors (common in urban canyons) cause perpendicular positional jumps. Our system mitigates this at two distinct layers:
> 1. **Kinematic Feature Layer:** Features like `heading_diff` and `speed_length_ratio` penalize candidate arcs whose orientation contradicts vehicular velocity vectors.
> 2. **Sequential Viterbi Transition Matrix:** The $10^{-5}$ disconnected transition penalty prevents the path from jumping to a parallel street, even if an errant GPS point lands directly on it.

### Q5: "What happens if a trip has no ground truth?"
> **Answer:**  
> The API handles unlabelled test trips gracefully. If `.route` is missing, `align_ground_truth_route()` returns `None`. The server skips accuracy scoring, sets `pointwise_top1_accuracy: "N/A"`, but still executes full candidate generation, ML inference, and HMM Viterbi decoding to reconstruct and return the continuous route coordinates for mapping.

---

## 7. Live Demonstration Checklist

Before calling the professor/evaluator over to your desk:

- [ ] **Laptop 1 Check:** Open PowerShell, verify `python src/server.py` is running and says `Uvicorn running on http://0.0.0.0:8000`.
- [ ] **IP Check:** Run `ipconfig` on Laptop 1; confirm you know your current IPv4 (e.g., `192.168.1.45`).
- [ ] **Laptop 2 Check:** Open browser to `http://<LAPTOP_1_IP>:8000/ui`. Verify `Server Status: Connected` (green badge).
- [ ] **Recommended Trip for Demo:** Select **`00000049`** in the dropdown.
- [ ] **Click "Run Evaluation":**
  - Point to Laptop 1 console: Show live logger receiving request, generating 26,420 candidates, and running Viterbi DP.
  - Point to Laptop 2 screen: Show the scorecard updating to **94.81% Accuracy**, **99.73% Continuity**, **+3.33% Gain**, and the interactive Folium map rendering the route.
- [ ] **Stress-Test Trip:** If asked for a difficult trip, select **`00000030`** (shows a huge **+20.83% continuity gain** proving the power of HMM Viterbi over raw classification).
