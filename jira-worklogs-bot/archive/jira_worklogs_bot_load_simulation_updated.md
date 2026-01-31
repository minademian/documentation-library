# 🧮 Load Simulation — JIRA Worklogs Bot Backend

## Context
This document models a theoretical load spike on the backend Java service handling JIRA Worklog submissions from Microsoft Teams.
It compares the steady weekday load with the recurring Friday spike (11:00–17:00).

---

## Assumptions

| Parameter | Symbol | Value | Description |
|------------|---------|--------|--------------|
| Engineers | E | 20 | Active engineers using the bot |
| Tasks per engineer per day | T | 10 | Average number of worklog entries per engineer per day |
| Working days | D | 5 | Monday–Friday |
| Spike window | — | 6 hours | 11:00 → 17:00 on Fridays |

---

## Equations

**Daily Submissions:**
$$R_{day} = E \times T$$

**Weekly Submissions:**
$$R_{week} = R_{day} \times D$$

**Peak-Hour Load (Friday spike period):**
$$R_{hour\_peak} = \frac{R_{day}}{6}$$

**Average Concurrent Requests (per second):**
$$R_{sec\_peak} = \frac{R_{hour\_peak}}{3600}$$

---

## Substituted Values

| Metric | Formula | Result |
|---------|----------|--------|
| Total daily submissions | R_day = 20 × 10 | **200 requests/day** |
| Total weekly submissions | R_week = 200 × 5 | **1,000 requests/week** |
| Peak-hour load (Friday 11–17) | R_hour_peak = 200 / 6 | **≈ 33.3 req/hour** |
| Peak concurrency | R_sec_peak = 33.3 / 3600 | **≈ 0.0093 req/sec** |

---

## Visualization

📊 *Weekly Load Context — including Friday concurrency overlay*

![Weekly Load Spike Visualization](./images/weekly_load_spike_fixed.png)

- **Blue–Purple lines:** Steady weekday load (Mon–Thu)
- **Red line:** Friday load spike (11–17h window)
- **Dashed black line:** Concurrent Friday requests per second (secondary axis)

---

## Interpretation
- Regular load Monday–Thursday is evenly distributed (~200 req/day).
- Friday sees concentrated activity late morning through afternoon.
- Peak ≈ 56 requests/hour (~0.016 req/sec).
- Backend can comfortably handle this within a single instance.

---

## Next Steps
- Validate assumptions once telemetry (Micrometer + Prometheus) is integrated.
- Adjust scaling thresholds if OAuth onboarding expands active user base.
- Integrate automated performance charting into CI/CD regression suite.
