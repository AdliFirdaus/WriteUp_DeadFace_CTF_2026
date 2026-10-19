# Finding Tampered and Fabricated Records with SQL

**Subtitle:** Two DeadFace CTF database challenges where the evidence of fraud wasn't a bad value — it was a broken relationship between tables.

Database forensics turned out to be one of my favourite parts of this CTF, because the queries map so cleanly onto the stories. Fraud in a database usually doesn't look like a single wrong number. It looks like a record that was changed when it shouldn't have been, or a record that exists without the other records that should accompany it. These two challenges — "After the Lock" and "Ghost in the Registry" — are one of each.

## Challenge 1 — After the Lock (150 pts): a record changed too late

Once a clinical trial is locked, its results are frozen, so any edit after the lock date is tampering. The task was to find the subject whose `primary_efficacy_delta` was modified after the Veltrexa study's lock date.

The trial data lived in `aurora_trials`, with `studies`, `subjects`, `results`, and an audit table `data_changes`. First I pulled the lock date, then looked for any primary-efficacy result modified after it:

```sql
SELECT data_lock_date FROM aurora_trials.studies;   -- 2025-12-01 (Veltrexa)

SELECT result_id, subject_id, value, modified_by, modified_at
FROM results
WHERE measure = 'primary_efficacy_delta'
  AND modified_at > '2025-12-01';
```

One row came back: result 200058, subject 137, changed on 2025-12-06 by `mreyes` — five days after the lock. Mapping the subject id to its code:

```sql
SELECT subject_code FROM subjects WHERE subject_id = 137;  -- VELT-0137
```

```
deadface{VELT-0137}
```

The whole case rests on one comparison: a `modified_at` timestamp against the recorded lock date. Nothing legitimate changes after the lock, so a later timestamp is the smoking gun.

## Challenge 2 — Ghost in the Registry (240 pts): a record with no history

Diverted narcotics need a paper trail that leads nowhere. The task was to find a patient who was issued at least one Schedule II prescription but had *never* been seen for a single clinical encounter, and report their MRN.

This one spans two databases: `aurora_pharmacy` (prescriptions and drugs, where `drugs.dea_schedule` marks Schedule II as `'II'`) and `aurora_ehr` (encounters, and the `patients` table that holds the MRN). They join on an enterprise `patient_id`. A real patient on controlled meds would have encounters behind those prescriptions; a fabricated one wouldn't. In SQL that's an anti-join:

```sql
SELECT DISTINCT pat.patient_id, pat.mrn, pat.first_name, pat.last_name
FROM aurora_pharmacy.prescriptions rx
JOIN aurora_pharmacy.drugs d  ON rx.drug_id = d.drug_id
JOIN aurora_ehr.patients pat  ON pat.patient_id = rx.patient_id
WHERE d.dea_schedule = 'II'
  AND NOT EXISTS (
        SELECT 1 FROM aurora_ehr.encounters e
        WHERE e.patient_id = rx.patient_id);
```

A single patient came back: id 1060, MRN `AH-0090001`, Dana Kessler — Schedule II scripts, zero encounters.

```
deadface{AH-0090001}
```

## Why these work

Both challenges reward thinking in relationships rather than values. In the first, integrity depends on the lock date, so comparing a modification timestamp against policy surfaces the violation directly. In the second, the fraud is a *missing* relationship — prescriptions with no matching encounters — which `NOT EXISTS` expresses cleanly and, unlike `NOT IN`, handles NULLs safely. The shared enterprise `patient_id` is what lets two separate systems (pharmacy and EHR) be correlated at all.

## What I took from it

Learning a schema fast is half the battle: `SHOW DATABASES` → `USE` → `SHOW TABLES` → `DESCRIBE`, and you know what you're working with. After that, two patterns cover a lot of database-forensics ground — comparing a timestamp against a policy date to catch after-the-fact edits, and using `NOT EXISTS` to find records that should have a partner but don't. Fraud often hides as an absence, not a bad value, and SQL is very good at finding absences.

(One small story detail I enjoyed: the ghost patient's surname, Kessler, is the same as `nkessler` from the network-forensics challenge. The series reuses names across categories, which makes it feel like one connected investigation.)

---

**Medium excerpt:** Fraud in a database rarely looks like a wrong number — it looks like a record changed too late, or one with no history behind it. Here are two DeadFace CTF SQL challenges solved with a timestamp check and an anti-join.

**Tags:** Cybersecurity · Capture The Flag · SQL · Digital Forensics · Infosec

**Cover image concept:** Two small table diagrams side by side — one with a row timestamped after a red "LOCKED" line, the other with a prescriptions row and a dangling arrow to an empty "encounters" table. Visualises "tampered" vs "fabricated" in one image.
