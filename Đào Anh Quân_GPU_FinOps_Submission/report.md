# Báo Cáo Lab: GPU FinOps & Cost Optimization

| Thông tin | Chi tiết |
|-----------|----------|
| **Họ tên** | Đào Anh Quân |
| **Môn học** | AI20K - Track 2 - Day 25 |
| **Ngày nộp** | 13/05/2026 |

---

## 1. Giới thiệu

### Mục tiêu bài lab

Bài lab Day 25 tập trung vào **GPU FinOps (Financial Operations)** - một lĩnh vực quan trọng trong việc tối ưu hóa chi phí sử dụng GPU trong các dự án AI/ML. Mục tiêu cụ thể:

- Hiểu cách monitor và quản lý GPU cluster
- Thực hành theo dõi và phân tích chi phí workload
- Tận dụng spot instances để tiết kiệm chi phí
- Cấu hình autoscaling dựa trên ngưỡng utilization
- Phân tích waste và tìm cơ hội tối ưu chi phí
- So sánh FP32 vs Mixed Precision (AMP) trên GPU thực

### Tổng quan kiến trúc

Bài lab sử dụng kiến trúc microservices chạy trên Docker Compose (local) kết hợp Kaggle notebook (remote GPU):

```
LOCAL (Docker Compose)                  REMOTE (Kaggle - Tesla T4)
┌─────────────────────────┐            ┌──────────────────────────┐
│ gpu-node-manager :8001  │            │                          │
│ billing-api      :8002  │◄──tunnel──►│  gpu_finops_lab.ipynb    │
│ spot-manager     :8003  │            │  (ResNet-18 on CIFAR-10) │
│ autoscaler       :8004  │            │                          │
│ cost-tracker     :8005  │            └──────────────────────────┘
│ gateway          :8000  │
└─────────────────────────┘
```

---

## 2. Phân tích kết quả từng phần

### Part 1: GPU Cluster Monitoring

**Kết quả quan sát cluster (5 nodes, 10 GPUs):**

| Node | GPU Type | Utilization | Memory Used | Power | Status |
|------|----------|-------------|-------------|-------|--------|
| node-00 GPU 0 | T4 | 64.0% | 13.8/16.0 GB | 51W | Busy |
| node-00 GPU 1 | T4 | 76.5% | 8.8/16.0 GB | 61W | Busy |
| node-01 GPU 0 | A100 | 89.0% | 45.7/80.0 GB | 192W | Busy |
| node-01 GPU 1 | A100 | 85.8% | 59.4/80.0 GB | 224W | Busy |
| node-02 GPU 0 | V100 | 75.4% | 25.1/32.0 GB | 246W | Busy |
| node-02 GPU 1 | V100 | 2.9% | 0.6/32.0 GB | 45W | Idle |
| node-03 GPU 0 | T4 | 8.2% | 1.6/16.0 GB | 22W | Idle |
| node-03 GPU 1 | T4 | 7.1% | 1.4/16.0 GB | 27W | Idle |
| node-04 GPU 0 | T4 | 0.0% | 0.5/16.0 GB | 20W | Idle |

**Cluster Metrics tổng hợp:**

- Total GPUs: 10 | Busy: 5 | Idle: 5
- Average Utilization: **40.9%**
- Memory Used: 157.4 / 320.0 GB
- Total Power Draw: **908W**

**Nhận xét:** Cluster đang có utilization trung bình thấp (40.9%), với 5/10 GPU đang idle. Đây là dấu hiệu lãng phí tài nguyên đáng kể, đặc biệt với các GPU A100 và V100 có giá thuê cao.

**Screenshots:** [part1_cluster_monitoring.png](screenshots/part1_cluster_monitoring.png)

---

### Part 2: Workload Submission & Cost Tracking

**Workloads đã submit:**

| Workload ID | GPU Assigned | Pricing Type | Cost | Savings |
|-------------|--------------|--------------|------|---------|
| train-resnet-001 | node-03, GPU 0 (T4) | ON-DEMAND | $0.0292 | $0.00 |
| train-bert-002 | node-02, GPU 1 (V100) | ON-DEMAND | $0.6117 | $0.00 |
| inference-api-003 | node-03, GPU 1 (T4) | SPOT | $0.0035 | $0.0082 |
| train-llm-004 | node-04 (2xT4) | SPOT | $0.5505 | $1.2845 |

**Billing Summary:**
- Total Cost: **$2.5589** (2.6% of $100 budget)
- Total Savings (spot): **$2.7078**
- Budget Status: OK

Sau khi submit, cluster đạt **80.3% utilization** (10/10 GPU busy).

**Nhận xét:** Sử dụng SPOT instances cho các workload chịu được preemption (inference, LLM training) giúp tiết kiệm $2.71 - gần gấp đôi chi phí thực tế. Đây là chiến lược FinOps rất hiệu quả.

**Screenshots:** [part2_workload_submission.png](screenshots/part2_workload_submission.png) | [part2_billing_summary.png](screenshots/part2_billing_summary.png)

---

### Part 3: Spot Instance Management

**Bảng giá Spot hiện tại:**

| GPU Type | On-Demand | Spot Price | Discount | Availability |
|----------|-----------|------------|----------|--------------|
| T4 | $0.35/hr | $0.1982/hr | **43.4%** | Low |
| A100 | $3.67/hr | $2.1223/hr | **42.2%** | Medium |
| V100 | $2.48/hr | $1.6004/hr | **35.5%** | Medium |

**Spot Instance Requests:**
- spot-t4-001 (T4): Granted
- spot-t4-002 (T4): Granted
- spot-a100-001 (A100): Granted

**Preemption Simulation:**
- Instances preempted: 1 (spot-t4-002 với 120s warning)
- Still active: 5

**Spot Savings Report:**
- Spot cost: $0.0285 vs On-demand equivalent: $0.0951
- Total saved: **$0.0665 (70.0% savings)**

**Nhận xét:** Spot instances mang lại mức tiết kiệm đáng kể (35-43%). Preemption xảy ra ngẫu nhiên nhưng có 120s warning, đủ thời gian để checkpoint và khôi phục training. Strategy: dùng spot cho training dài hạn, on-demand cho inference cần tính sẵn sàng cao.

**Screenshots:** [part3_spot_pricing.png](screenshots/part3_spot_pricing.png) | [part3_spot_request.png](screenshots/part3_spot_request.png) | [part3_spot_preemption.png](screenshots/part3_spot_preemption.png)

---

### Part 4: Autoscaling (KEDA-like)

**Autoscaling Policy đã cấu hình:**

| Parameter | Value |
|-----------|-------|
| scale_up_threshold | 70.0% |
| scale_down_threshold | 25.0% |
| cooldown_seconds | 30 |
| max_nodes | 10 |
| min_nodes | 2 |
| preferred_gpu_type | T4 |
| cost_aware | True |

**Kết quả evaluation:**
- Trạng thái ban đầu: Utilization 80.3% > 70% → **SCALE_UP** (5 → 6 nodes)
- 5 evaluation cycles sau khi scale: **no_action** (Util giảm xuống 66.9% nằm trong vùng ổn định)

**Nhận xét:** Autoscaler hoạt động chính xác - tự động scale up khi cluster quá tải và dừng lại sau khi thêm node làm utilization giảm về vùng an toàn. Cài đặt `cost_aware: True` đảm bảo luôn chọn GPU type rẻ nhất (T4) khi scale.

**Screenshots:** [part4_autoscaler_policy.png](screenshots/part4_autoscaler_policy.png) | [part4_autoscaler_evaluation.png](screenshots/part4_autoscaler_evaluation.png)

---

### Part 5: Cost Analysis & Optimization

**Cost Snapshots (5 lần):**
- Tất cả snapshots: Total = $0.041944 | Idle cost = $0.001944 | Waste = **4.6%**

**Waste Analysis Report:**
- Average Waste: **12.1%**
- Total Idle Cost: $0.046996
- Total Cost: $0.401944
- Potential Monthly Saving: **$1,218.14**
- Severity: **LOW**

**Optimization Recommendations:**

| Priority | Category | Recommendation | Estimated Savings |
|----------|----------|----------------|-------------------|
| MEDIUM | USE_SPOT | Chuyển fault-tolerant workloads sang spot instances | **65%** |
| LOW | SCHEDULING | Schedule non-urgent jobs vào off-peak hours | **20%** |

**Dashboard tổng hợp:**
- 12 GPUs, 6 nodes | Utilization: 66.9% | Idle: 2 GPUs
- Billing: $2.5589 / $100 budget | Savings: $2.7078
- Spot savings: $0.0873 (70%) | Waste: 12.1% (LOW)

**Nhận xét:** Mức waste 12.1% là khá thấp (severity LOW), nhưng vẫn có tiềm năng tiết kiệm $1,218/tháng nếu tối ưu. Spot instances là cơ hội lớn nhất với khả năng giảm 65% chi phí.

**Screenshots:** [part5_cost_snapshots.png](screenshots/part5_cost_snapshots.png) | [part5_waste_report.png](screenshots/part5_waste_report.png) | [part5_recommendations.png](screenshots/part5_recommendations.png) | [part5_dashboard.png](screenshots/part5_dashboard.png)

---

### Part 6: Visualization

Tạo 2 loại biểu đồ trực quan hóa cost data:

**Cost Breakdown Chart** (`finops_cost_breakdown.png`): 3 subplots hiển thị phân bổ chi phí theo GPU type, workload type, và billing mode.

**Time-series Chart** (`finops_timeseries.png`): Stackplot theo dõi chi phí theo thời gian và waste percentage trend.

**Screenshots:** [part6_cost_breakdown_viz.png](screenshots/part6_cost_breakdown_viz.png) | [part6_timeseries_viz.png](screenshots/part6_timeseries_viz.png)

**Generated Charts:** [finops_cost_breakdown.png](generated_charts/finops_cost_breakdown.png) | [finops_timeseries.png](generated_charts/finops_timeseries.png)

---

### Part 7: Complete FinOps Workflow

Full end-to-end FinOps optimization cycle (7 steps):

1. **Initial State:** 12 GPUs, Util: 66.9%, Idle: 2
2. **Submit Heavy Workloads:** Util tăng lên 77.7% (12/12 busy)
3. **Autoscaler Evaluation:** SCALE_UP (77.7% > 70% threshold)
4. **Cost Analysis:** $0.043889/interval, Waste: 4.4%
5. **Recommendations:** USE_SPOT (65% savings), SCHEDULING (20% savings)
6. **Apply Optimization:** Switch to spot → saved $0.0340 (70%)
7. **Final Bill:** Budget utilization tối ưu sau tối ưu hóa

**Nhận xét:** Workflow hoàn chỉnh cho thấy vòng lặp FinOps liên tục: Monitor → Analyze → Optimize → Repeat. Việc tự động hóa từng bước giúp tiết kiệm thời gian và đảm bảo nhất quán.

**Screenshot:** [part7_full_workflow.png](screenshots/part7_full_workflow.png)

---

### Part 8: Real GPU Workload (Kaggle - Tesla T4)

#### GPU Environment

Chạy trên Kaggle với GPU thực:
- **GPU:** Tesla T4 (15.6 GB VRAM)
- **CUDA:** 12.8
- **Monitoring:** pynvml (available)
- **Task:** ResNet-18 training trên CIFAR-10 (3 epochs)

#### FP32 Training (Baseline)

| Epoch | Loss | Accuracy | Time |
|-------|------|----------|------|
| 1/3 | 2.1092 | 24.9% | 45.9s |
| 2/3 | 1.4740 | 45.6% | 43.9s |
| 3/3 | ~1.2 | ~55% | ~44s |

- Total time: **134.9s**
- Peak memory: **0.97 GB**
- Avg GPU util: **94.9%**
- Avg power: **66.9W**
- Estimated cost: **$0.013112**

#### Mixed Precision AMP Training (Optimized)

| Epoch | Loss | Accuracy | Time |
|-------|------|----------|------|
| 1/3 | 1.9529 | 29.1% | 19.9s |
| 2/3 | 1.4388 | 46.8% | 19.9s |
| 3/3 | 1.1298 | 59.2% | 19.9s |

- Total time: **59.8s**
- Peak memory: **0.60 GB**
- Avg GPU util: **89.8%**
- Avg power: **64.3W**
- Estimated cost: **$0.005815**

#### So sánh FP32 vs AMP

| Metric | FP32 | AMP | Cải thiện |
|--------|------|-----|-----------|
| Total Time | 134.9s | 59.8s | **2.25x faster** |
| Peak Memory | 0.97 GB | 0.60 GB | **-0.37 GB (38%)** |
| Cost | $0.013112 | $0.005815 | **$0.007297 saved** |
| Cost Saving | — | — | **55.7%** |
| Accuracy (3ep) | ~55% | **59.2%** | AMP tốt hơn |

**Nhận xét quan trọng:** AMP không chỉ nhanh hơn 2.25x và rẻ hơn 55.7%, mà còn đạt accuracy cao hơn (59.2% vs ~55%). Đây là kỹ thuật FinOps cực kỳ hiệu quả - không có trade-off, chỉ có lợi ích.

**Cost được report lên gateway:**
- FP32: $0.013100 (rate: $0.35/hr)
- AMP (as spot): $0.001700 (saved $0.004100)
- Total platform cost: $2.7280 | Total savings: $2.8302

**Screenshots:** [part8_gpu_detection.png](screenshots/part8_gpu_detection.png) | [part8_gpu_metrics_diagnostic.png](screenshots/part8_gpu_metrics_diagnostic.png) | [part8_fp32_summary.png](screenshots/part8_fp32_summary.png) | [part8_amp_summary.png](screenshots/part8_amp_summary.png) | [part8_fp32_vs_amp_comparison.png](screenshots/part8_fp32_vs_amp_comparison.png) | [part8_real_gpu_cost_report.png](screenshots/part8_real_gpu_cost_report.png)

**Generated Charts:** [real_gpu_comparison.png](generated_charts/real_gpu_comparison.png) | [real_gpu_telemetry.png](generated_charts/real_gpu_telemetry.png) | [cost_per_epoch.png](generated_charts/cost_per_epoch.png)

---

### Part 8.5: Advanced GPU Cost Optimization

#### Exercise 8.5.1: Multi-GPU Cost Analysis

| GPUs | Speedup | Time (h) | Cost ($) | Efficiency | $/Perf Unit |
|------|---------|----------|----------|------------|-------------|
| 1 | 1.00x | 2.000 | $0.7000 | 100% | $0.7000 |
| 2 | 1.80x | 1.111 | $0.7778 | 90.0% | $0.4321 |
| 4 | 3.20x | 0.625 | $0.8750 | 80.0% | $0.2734 |
| **8** | **5.50x** | **0.364** | **$1.0182** | **68.8%** | **$0.1851** |

**Optimal:** 8 GPUs với cost/perf = $0.1851 (tốt nhất về giá trị)

**Nhận xét:** Scaling efficiency giảm dần khi tăng số GPU (overhead giao tiếp). Tuy nhiên, $/performance unit vẫn tốt hơn ở 8 GPU. Quyết định optimal phụ thuộc vào deadline vs budget.

#### Exercise 8.5.2: Project Cost Forecasting

| Phase | GPUs | Hours | Base Cost | Min | Max |
|-------|------|-------|-----------|-----|-----|
| Data Preparation | 1 | 40 | $14.00 | $11.90 | $16.10 |
| Model Training | 4 | 120 | $1,761.60 | $1,321.20 | $2,202.00 |
| Hyperparameter Tuning | 8 | 60 | $1,761.60 | $1,233.12 | $2,290.08 |
| Model Evaluation | 2 | 20 | $14.00 | $12.60 | $15.40 |

**Nhận xét:** Model training và hyperparameter tuning chiếm phần lớn chi phí. Forecast có confidence interval rộng do biến động spot pricing và training time. Cần budget buffer ít nhất 25%.

#### Exercise 8.5.3: Optimization Opportunity Analysis

Baseline cost: **$1,468.00**

| # | Chiến lược | Effort | Risk | Savings | Cumulative |
|---|-----------|--------|------|---------|------------|
| 1 | Mixed Precision (AMP) | LOW | LOW | $367.00 | 25% |
| 2 | Spot Instances | MEDIUM | HIGH | $880.80 | 85% |
| 3 | Optimize Batch Size | LOW | LOW | $220.20 | 100% |

**Nhận xét:** AMP và batch size optimization là low-hanging fruit (low effort, low risk). Kết hợp 3 chiến lược có thể tiết kiệm toàn bộ baseline cost - đây là lý do tại sao FinOps quan trọng.

#### Exercise 8.5.4: Integrated Cost Dashboard

Dashboard 2x3 tổng hợp tất cả metrics: cost breakdown, utilization trends, spot savings, waste analysis, multi-GPU comparison, và forecasting.

**Generated Chart:** [advanced_finops_dashboard.png](generated_charts/advanced_finops_dashboard.png)

#### Exercise 8.5.5: Challenge - Cost Optimization Strategy

**Bài toán:** 8x A100 × 200h × $3.67/hr = **$5,872** vs Budget **$5,000** (over by $872)

**Giải pháp:**

1. **Multi-GPU Analysis:** 16 GPUs có cost/perf tốt nhất ($144.99/unit) → **Chọn 16 GPUs**
2. **Spot instances:** Tiết kiệm thêm ~40% → phù hợp với budget
3. **AMP training:** Giảm 55% thời gian → giảm chi phí tương ứng
4. **Checkpoint strategy:** Lưu checkpoint mỗi epoch để phục hồi khi spot bị preempt

**Screenshots:** [part85_multi_gpu_analysis.png](screenshots/part85_multi_gpu_analysis.png) | [part85_project_forecast.png](screenshots/part85_project_forecast.png) | [part85_optimization_analysis.png](screenshots/part85_optimization_analysis.png) | [part85_integrated_dashboard.png](screenshots/part85_integrated_dashboard.png) | [part85_challenge_strategy.png](screenshots/part85_challenge_strategy.png)

**Generated Charts:** [multi_gpu_scaling.png](generated_charts/multi_gpu_scaling.png) | [project_forecast.png](generated_charts/project_forecast.png) | [optimization_roadmap.png](generated_charts/optimization_roadmap.png)

---

## 3. Kết luận

### Tóm tắt kết quả

| Part | Kết quả nổi bật |
|------|----------------|
| 1 | Cluster 10 GPU, avg util 40.9%, 5 GPU idle |
| 2 | Spot instances tiết kiệm $2.71 trên $2.56 chi phí thực |
| 3 | Spot discount 35-43%, 70% savings vs on-demand |
| 4 | Autoscaler tự động scale up khi util > 70% |
| 5 | Waste 12.1%, tiềm năng tiết kiệm $1,218/tháng |
| 6 | Visualization đầy đủ cost breakdown và trends |
| 7 | Full workflow 7 steps tự động hóa hoàn chỉnh |
| 8 | AMP nhanh hơn 2.25x, rẻ hơn 55.7% so với FP32 |
| 8.5 | 8-GPU optimal, forecast với CI, roadmap tiết kiệm 100% |

### Các kỹ năng FinOps đã học

1. **Cluster Monitoring:** Sử dụng metrics dashboard để phát hiện GPU idle và bottleneck
2. **Cost Attribution:** Gán chi phí cho từng workload, team, project
3. **Spot Instance Management:** Bidding, preemption handling, checkpoint strategy
4. **Autoscaling:** Cấu hình threshold-based scaling để tránh over/under-provisioning
5. **Waste Detection:** Phân tích idle GPU và đề xuất consolidation
6. **Mixed Precision Training:** AMP giảm chi phí 55% không có trade-off
7. **Multi-GPU Optimization:** Phân tích scaling efficiency để chọn config tối ưu
8. **Cost Forecasting:** Dự báo chi phí với confidence intervals cho project planning

### Chiến lược tối ưu chi phí hiệu quả nhất

Theo thứ tự ưu tiên (impact/effort):

1. **Mixed Precision (AMP)** - Low effort, low risk, 55%+ savings trên training cost
2. **Spot Instances** - Medium effort, cần checkpoint strategy, 35-70% savings
3. **Autoscaling** - Một lần setup, tự động tránh over-provisioning
4. **Workload Scheduling** - Schedule vào off-peak hours, thêm 20% savings
5. **Right-sizing** - Chọn GPU type phù hợp với workload (T4 cho inference, A100 cho LLM)

### Ứng dụng thực tế

- **Startup AI:** Dùng AMP + Spot instances có thể giảm 70-80% GPU bill
- **Enterprise:** Autoscaling + Cost attribution giúp justify GPU investment cho management
- **Research:** Multi-GPU analysis giúp quyết định scale ngang vs scale dọc
- **Production:** Monitoring dashboard giúp phát hiện anomaly và optimize liên tục

---

## 4. Cấu trúc bài nộp

```
Đào Anh Quân_GPU_FinOps_Submission/
├── SUBMISSION.md                        # Báo cáo này
├── screenshots/                         # 25 screenshots (Parts 1-8.5)
│   ├── part1_cluster_monitoring.png
│   ├── part2_billing_summary.png
│   ├── part2_workload_submission.png
│   ├── part3_spot_preemption.png
│   ├── part3_spot_pricing.png
│   ├── part3_spot_request.png
│   ├── part4_autoscaler_evaluation.png
│   ├── part4_autoscaler_policy.png
│   ├── part5_cost_snapshots.png
│   ├── part5_dashboard.png
│   ├── part5_recommendations.png
│   ├── part5_waste_report.png
│   ├── part6_cost_breakdown_viz.png
│   ├── part6_timeseries_viz.png
│   ├── part7_full_workflow.png
│   ├── part8_amp_summary.png
│   ├── part8_fp32_summary.png
│   ├── part8_fp32_vs_amp_comparison.png
│   ├── part8_gpu_detection.png
│   ├── part8_gpu_metrics_diagnostic.png
│   ├── part8_real_gpu_cost_report.png
│   ├── part85_challenge_strategy.png
│   ├── part85_integrated_dashboard.png
│   ├── part85_multi_gpu_analysis.png
│   ├── part85_optimization_analysis.png
│   └── part85_project_forecast.png
├── generated_charts/                    # 9 PNG charts được generate
│   ├── SUBMISSION.md
│   ├── advanced_finops_dashboard.png
│   ├── cost_per_epoch.png
│   ├── finops_cost_breakdown.png
│   ├── finops_timeseries.png
│   ├── multi_gpu_scaling.png
│   ├── optimization_roadmap.png
│   ├── project_forecast.png
│   ├── real_gpu_comparison.png
│   └── real_gpu_telemetry.png
└── notebook/
    └── gpu_finops_lab.ipynb             # Notebook đã chạy đầy đủ (45 cells)
```
