# SQL Murder Mystery Project

## Project Overview

[#project-overview](#project-overview)

**Project Title**: SQL Murder Mystery
**Level**: Intermediate
**Database**: `sql_murder_mystery` (PostgreSQL)

This project demonstrates SQL investigative querying skills using the classic "SQL Murder Mystery" dataset (built around a fictional crime database for SQL City). Starting from a single clue — a murder on January 15, 2018, in SQL City — the goal is to progressively narrow down witnesses, cross-reference clues, and identify a suspect using nothing but SQL queries against a relational database. The project goes a step further with a bonus challenge to uncover the mastermind behind the crime in 22 queries or fewer.

## Objectives

[#objectives](#objectives)

1. **Query a relational crime database**: Navigate multiple linked tables (`crime_scene_report`, `person`, `interview`, `get_fit_now_member`, `get_fit_now_check_in`, `drivers_license`, `facebook_event_checkin`) to trace a case from a single lead to a solved crime.
2. **Filtering & Pattern Matching**: Use `WHERE`, `LIKE`, and `ORDER BY` to isolate relevant records from noisy tables (e.g. narrowing down witnesses, matching partial plate numbers).
3. **CTAS (Create Table As Select)**: Use `CREATE TABLE ... AS` to persist intermediate suspect lists for easier cross-referencing.
4. **Joins**: Use `LEFT JOIN` to combine `drivers_license` and `person` data to attach identities to physical/vehicle descriptions.
5. **Multi-clue Cross-referencing**: Combine testimony, gym membership records, check-in logs, and license data to eliminate suspects and confirm a single match.
6. **Bonus Challenge**: Re-interview the confirmed suspect to extract a secondary clue, then filter on physical description, vehicle, and event attendance to identify the mastermind behind the murder.

## Project Structure

[#project-structure](#project-structure)

### 1. Retrieve the Crime Scene Report

[#1-retrieve-the-crime-scene-report](#1-retrieve-the-crime-scene-report)

**Objective**: Pull the report matching a murder on January 15, 2018 in SQL City.

```sql
SELECT *
FROM crime_scene_report
WHERE crime_type = 'murder'
AND DATE = 20180115
AND city = 'SQL City';
```

The report reveals security footage showing two witnesses: one who lives at the *last house on Northwestern Dr*, and one named *Annabel* who lives on *Franklin Ave*.

### 2. Identify the Witnesses

[#2-identify-the-witnesses](#2-identify-the-witnesses)

**Task**: Find the first witness — the last house on Northwestern Dr (i.e. the highest address number on that street).

```sql
SELECT * FROM person
WHERE address_street_name = 'Northwestern Dr'
ORDER BY address_number DESC;
```
> Result: **Morty Schapiro** — ID 14887, address 4919 Northwestern Dr

**Task**: Find the second witness — Annabel, on Franklin Ave.

```sql
SELECT * FROM person
WHERE name LIKE 'Annabel%'
AND address_street_name = 'Franklin Ave';
```
> Result: **Annabel Miller** — ID 16371, address 103 Franklin Ave

### 3. Review Witness Interviews

[#3-review-witness-interviews](#3-review-witness-interviews)

**Objective**: Pull both witnesses' interview transcripts.

```sql
SELECT * FROM interview
WHERE person_id IN (14887, 16371);
```

- **Morty** described a man with a "Get Fit Now Gym" bag, a gold membership starting with "48Z", and a getaway car with a plate containing "H42W".
- **Annabel** recognized the killer from her gym, where she'd seen him working out on January 9th.

### 4. Investigate the Gym Membership Clue

[#4-investigate-the-gym-membership-clue](#4-investigate-the-gym-membership-clue)

**Objective**: Find gold members with a membership ID starting with "48Z".

```sql
SELECT * FROM get_fit_now_member
WHERE id LIKE '48Z%'
AND membership_status = 'gold';
```

**CTAS**: Persist the candidates as a working suspects table.

```sql
CREATE TABLE Suspects AS (
    SELECT * FROM get_fit_now_member
    WHERE id LIKE '48Z%'
    AND membership_status = 'gold'
);

SELECT * FROM Suspects;
```

> Two suspects surfaced: **Joe Germuska** (48Z7A) and **Jeremy Bower** (48Z55), both gold members.

### 5. Cross-reference Check-in Records

[#5-cross-reference-check-in-records](#5-cross-reference-check-in-records)

**Objective**: Confirm which suspect(s) checked in at the gym on January 9th, per Annabel's account.

```sql
SELECT * FROM get_fit_now_check_in
WHERE check_in_date = 20180109
AND membership_id IN ('48Z7A', '48Z55');
```

> Both suspects checked in on the 9th — this clue alone doesn't eliminate anyone, so the getaway car becomes the deciding clue.

### 6. Trace the Getaway Car

[#6-trace-the-getaway-car](#6-trace-the-getaway-car)

**Objective**: Search driver's licenses for a plate number containing "H42W".

```sql
SELECT * FROM drivers_license
WHERE plate_number LIKE 'H42W%'
OR plate_number LIKE '%H42W%'
OR plate_number LIKE '%H42W';
```

**Join to identities**: The `drivers_license` table has no names, so join it to `person` on license ID.

```sql
CREATE TABLE Full_Data AS (
    SELECT dl.age, dl.height, dl.hair_color, dl.gender, dl.plate_number,
           dl.car_make, dl.car_model, p.name, p.ssn, p.address_street_name, p.id
    FROM drivers_license AS dl
    LEFT JOIN person AS p
    ON dl.id = p.license_id
);

SELECT * FROM Full_Data
WHERE plate_number LIKE 'H42W%'
OR plate_number LIKE '%H42W%'
OR plate_number LIKE '%H42W';
```

### Solution: The Killer

[#solution-the-killer](#solution-the-killer)

Cross-referencing the plate number against the joined identity table narrows the match down to a single suspect:

> **The killer is Jeremy Bower.**

## Bonus Challenge: Finding the Mastermind

[#bonus-challenge-finding-the-mastermind](#bonus-challenge-finding-the-mastermind)

Solving the murder isn't the end of the case — Jeremy Bower's own interview transcript reveals he was hired by someone else. Completed in the query budget of 22 or fewer.

**Task**: Pull Jeremy Bower's interview.

```sql
SELECT * FROM interview
WHERE person_id = 67318;
```

> Jeremy described a woman: 5'5"–5'7", red hair, drives a Tesla Model S, and attended the SQL Symphony Concert three times in October 2017.

**Task**: Filter the joined identity table on her physical and vehicle description.

```sql
SELECT * FROM Full_Data
WHERE height BETWEEN 65 AND 67
AND hair_color = 'red'
AND gender = 'female'
AND car_make = 'Tesla'
AND car_model = 'Model S';
```

**Task**: Confirm concert attendance for the remaining candidates via Facebook event check-ins.

```sql
SELECT * FROM facebook_event_checkin
WHERE person_id IN (78881, 90700, 99716);
```

> Only ID **99716** attended the concert three times in October 2017.

**Task**: Confirm identity.

```sql
SELECT * FROM Full_Data
WHERE id = 99716;
```

> **The mastermind behind the murder is Miranda Priestly.**

## Findings

[#findings](#findings)

- The crime scene report and witness interviews provided the initial thread (gym bag, membership prefix, partial plate number).
- CTAS was used to snapshot a working suspect pool (`Suspects`) rather than re-running the same filter repeatedly.
- A `LEFT JOIN` between `drivers_license` and `person` was necessary because vehicle/physical data and identity data lived in separate tables with no direct name field.
- The getaway car's partial plate number ("H42W") was the deciding clue that isolated Jeremy Bower from Joe Germuska, since both suspects otherwise matched the gym and check-in clues.
- The bonus round required treating the confirmed killer as a new lead rather than an endpoint — his interview reopened the case and pointed to the actual instigator.

## Conclusion

[#conclusion](#conclusion)

This project demonstrates core investigative SQL skills: filtering with `WHERE` and `LIKE`, sorting with `ORDER BY`, persisting intermediate results with CTAS, and joining across tables to reconcile identity, vehicle, and behavioral data. It also shows how to work a case incrementally — each query either eliminates a candidate or adds a new clue — mirroring how SQL is used in real investigative and analytical workflows.

## How to Use

[#how-to-use](#how-to-use)

1. **Get the database**: The SQL Murder Mystery schema and data are publicly available (originally built by Knight Lab) — load the `.sql` schema/data files into a PostgreSQL instance named e.g. `sql_murder_mystery`.
2. **Run the investigation queries**: Execute the queries in this README (or the accompanying `.sql` file) in order, starting from the crime scene report.
3. **Verify your answer**: Use an `INSERT` statement with your suspect's ID against the game's validation table (per the mystery's built-in check) to confirm the correct killer and mastermind.
4. **Explore further**: Try re-solving the case in fewer queries, or extend the investigation to see what other clues the database supports.

## Author — [Your Name]

[#author-your-name](#author-your-name)

This project showcases SQL skills essential for investigative querying, filtering, joins, and CTAS-based data analysis. For more content on SQL and data analysis, connect with me through the following channels:

- **LinkedIn**: [Add your LinkedIn URL]
- **GitHub**: [Add your GitHub profile URL]
- **Portfolio**: [Add your portfolio site]

Thank you for checking out this project!
