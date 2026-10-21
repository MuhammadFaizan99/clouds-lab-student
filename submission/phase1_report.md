# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
49
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->243.0<!--/answer--> | <!--answer:vm_warm_median_ms-->211.2<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2043.6<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->643.0<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1506.1<!--/answer--> | <!--answer:function_warm_median_ms-->634.2<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Warm latency distribution (20 requests per platform)
Each # represents one request.

Latency (ms)      VM       Cloud Run    Function
100-199           0        #            ######
200-299           ####################  0  0
300-599           0        0            0
600-649           0        ###########  #########
650-699           0        ########     #####

Warm median:
VM: 211.2 ms
Cloud Run: 643.0 ms
Cloud Function: 634.2 ms
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
For the IaaS VM, requests are handled by the existing Docker container. No additional VMs or containers are created automatically. Its capacity is limited by the selected e2-micro machine (shared CPU and 1 GB RAM). At 100 concurrent requests, performance may degrade. Cloud Run automatically creates additional container instances as traffic increases, up to the --max-instances=3 limit we configured. Cloud Functions Gen 2 also automatically scales by creating function instances, with the same --max-instances=3 limit. Both serverless platforms can scale to zero when idle.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The IaaS VM required the most manual work. I created a custom VPC, subnet, firewall rules, and an e2-micro VM, installed Docker, configured Artifact Registry authentication, pulled the image, and started the container manually. I am responsible for maintaining the VM and its software. Cloud Run required less effort because I only deployed the existing Docker image, configured port 8080, public access, and scaling limits. Google manages the underlying infrastructure. Cloud Functions required deploying Python source code using main.py, specifying the Python 3.11 runtime, HTTP trigger, entry point, and scaling limits. Google handles infrastructure and runtime management.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-338236534079.us-central1.run.app
LAST DEPLOYED BY: service-338236534079@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-20T19:19:54.160445Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-338236534079.us-central1.run.app
LAST DEPLOYED BY: Muhammad.Faizan@eurecom.fr
LAST DEPLOYED AT: 2026-10-20T19:12:51.286945Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output confirms that both my Cloud Function (echo) and Cloud Run application (echo-bot) run as Cloud Run services. The Function was deployed by Google's Cloud Functions administrative service account, while echo-bot was deployed directly by me. This shows that second-generation Cloud Functions use Cloud Run infrastructure behind the scenes. Both platforms run containerized workloads, automatically manage instances, and can scale down when idle. Therefore, their cold-start times are similar: 1506.1 ms for the Function and 2043.6 ms for Cloud Run. The main difference is deployment: for Cloud Run, I built and pushed a Docker image, while for Cloud Functions, I provided Python source code and an HTTP entry point. Google automatically builds and manages the Function's container.
<!--/answer-->