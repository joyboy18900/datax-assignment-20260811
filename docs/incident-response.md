# Incident Response: Rejection Rate Spike

On-call runbook for the `HighRejectionRate` and `RejectionRateSpike` alerts
(`prometheus/alert-rules.yml`). Assumes no prior familiarity with this system.

## 1. Initial Triage

1. Check which alert fired - it sets the urgency:
   - `RejectionRateSpike` (critical) - 5m rate is 3x+ the 30m trailing baseline. Treat as active and ongoing.
   - `HighRejectionRate` (warning) - sustained above 30% for 10+ minutes. Elevated, not necessarily urgent.
2. Confirm the API itself is up: `curl http://localhost:8080/healthz`. If this fails, or `AgentAPIDown` is also firing, this is a bigger outage - go straight to the escalate row in section 3.
3. Open Grafana (`localhost:3000`, admin/admin), "Agent API Monitoring" dashboard. Check "Overall Rejection Rate" (current level) and "Rejections by Reason" (which category is driving it).

## 2. Investigation

```bash
# Rejection rate by reason, last 5m - concentrated in one category, or spread across all?
curl -s -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=sum(rate(agent_rejections_total[5m])) by (reason)'

# Was there a recent deploy? deployment/manifest.yml is stamped with the
# commit SHA on every merge to main (.github/workflows/ci.yml, deploy job).
git log -3 -- deployment/manifest.yml

docker compose logs agent-api --tail 100
```

Known false positive: `RejectionRateSpike` compares the last 5m to a 30m
trailing baseline. If `agent-api` restarted in the last ~30 minutes, that
baseline has no real history yet and the alert can fire on normal traffic.
Check uptime before treating it as a real incident:

```bash
curl -s -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=time()-process_start_time_seconds{job="agent-api"}'
```

## 3. Decision Framework

| Observation | Likely cause | Action |
|---|---|---|
| Spike concentrated in one `reason`, API healthy | Coordinated probing/attack attempt - system is rejecting it correctly | No action, this is working as designed. Monitor. |
| Spike spread across all 3 reasons, including messages that look legitimate | A recent change to `classify_rejection()` in `agent-api/app.py` became too aggressive | Mitigate: roll back to the previous commit SHA in `deployment/manifest.yml`, redeploy |
| Matches an `agent-api` restart in the last ~30m, alert is specifically `RejectionRateSpike` | Cold-start baseline artifact | No action, re-observe once 30m of uptime has accumulated |
| `AgentAPIDown` also firing, rate keeps climbing after mitigation, or `make eval` golden-dataset accuracy drops (real users being blocked) | Broader outage, or mitigation did not work | Escalate, page a second engineer |

## 4. Post-Incident Actions

1. Confirm `deployment/manifest.yml`'s `image_tag` matches the commit that fixed it (the CD job stamps this automatically on merge to main).
2. Run `make eval`, confirm gates pass (90% golden accuracy, 5% max golden rejection, 60% min adversarial rejection).
3. Write a short postmortem: trigger, reason breakdown at peak, time to detect and mitigate.
