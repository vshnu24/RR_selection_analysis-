# SupeRR Selector – Pressure Performance Analysis

## 1. Project Overview

The **SupeRR Selector** project identifies batters and bowlers who perform effectively in high-pressure situations in T20 cricket.

The analysis uses the supplied ball-by-ball and supporting match datasets to:

1. Clean and prepare the cricket data.
2. Identify high-pressure deliveries using match context.
3. Compare each player's pressure performance with their overall baseline.
4. Calculate performance scores and rank eligible players.
5. Produce two ordered lists: the Top 10 batters and Top 10 bowlers.

The rankings are intended to be evidence-based and reproducible. They are not based on player reputation, and they do not predict future performance or guarantee suitability for a particular team.

## 2. Dataset Description

The project uses the following six CSV files:

| Dataset | Purpose |
|---|---|
| `ball_by_ball.csv` | Delivery-level information, runs, players, wickets and innings |
| `matches.csv` | Match details, innings limits and revised targets |
| `competitions.csv` | Competition information |
| `competition_team_player_mapping.csv` | Links players and teams to competitions |
| `playing_eleven.csv` | Match squad and playing-eleven information |
| `teams.csv` | Team details |

The ball-by-ball dataset is the primary source for calculating player performance. The supporting datasets provide match, competition, team and player context.

**Important:** The exact column names, data types, extras codes, dismissal labels and revised-innings fields must be checked against the supplied files before the analysis is treated as final.



## 3. Methodology and Workflow

### 3.1 Data Cleaning – `data_cleaning.ipynb`

The first stage prepares the supplied datasets for analysis.

Operations include:

- Loading all six CSV files using Pandas.
- Removing duplicate rows.
- Removing unnecessary spaces from column names and text values.
- Checking missing values.
- Checking the data types of important columns.

The purpose of this stage is to improve data consistency and make later calculations more reliable.

**Validation checks:**

- Confirm that all six input files load successfully.
- Check duplicate records and missing values.
- Confirm that match IDs and player IDs have consistent data types.
- Check that the delivery records can be linked to the correct matches.

Cleaning should not silently replace important missing values or remove valid cricket events. Any exclusions or corrections should be documented.

### 3.2 Pressure Situation Identification – `pressure_analysis.ipynb`

A delivery is classified as a pressure delivery when at least one of the following model conditions is met.

| Condition | Pressure definition |
|---|---|
| Late innings | Over 16 or later |
| Wickets lost | At least 6 wickets have fallen before the delivery |
| Difficult chase | Required run rate is at least 10 runs per over |
| Few wickets remaining | At most 4 wickets remain and at most 60 legal balls remain in the chase |
| Final stage of chase | At most 30 legal balls remain and required run rate is at least 9 |

The pressure state is calculated **before the delivery**, so the delivery being evaluated does not change the match situation used to classify itself.

#### Required Run Rate

For a second-innings chase:

\[
\text{Runs Needed} = \text{Target} - \text{Runs Scored Before Delivery}
\]

\[
\text{Required Run Rate} =
\frac{\text{Runs Needed}\times 6}
{\text{Legal Balls Remaining}}
\]

The target is taken from the revised target field when it is valid and available. Otherwise, it is calculated as the first-innings total plus one.

#### Balls Remaining

\[
\text{Balls Remaining} =
\text{Maximum Legal Balls in Innings}
-
\text{Legal Balls Bowled Before Delivery}
\]

The innings limit should use the applicable revised overs when supplied and validated. Wides and no-balls do not count as legal balls.

#### Pressure Classification

A delivery is labelled `pressure = True` if any one of the five conditions is satisfied. Otherwise, it is labelled `pressure = False`.

This definition is a modelling choice for this project, not an official universal cricket definition. The thresholds should be tested for sensitivity before final submission.

### 4.3 Batter Performance Ranking – `batter_ranking.ipynb`

Batter performance is measured in pressure situations and compared with the same batter's overall performance in the analysed dataset.

#### Metrics

**A. Pressure Runs**

Total batter runs scored on deliveries classified as pressure deliveries.

**B. Pressure Balls**

Number of balls faced by the batter in pressure situations. Wides are excluded from balls faced. The exact treatment of no-balls and other extras must follow the source data's delivery-level definitions.

**C. Pressure Strike Rate**

\[
\text{Pressure Strike Rate} =
\frac{\text{Pressure Runs}}
{\text{Pressure Balls}}\times 100
\]

**D. Pressure Batting Average**

\[
\text{Pressure Average} =
\frac{\text{Pressure Runs}}
{\text{Pressure Dismissals}}
\]

A batter dismissal should be charged to the batter only when the dismissed player is the striker and the dismissal type is correctly represented in the data. Non-striker run-outs must not be counted as striker dismissals.

**E. Strike Rate Uplift**

\[
\text{SR Uplift} =
\text{Pressure Strike Rate}
-
\text{Overall Strike Rate}
\]

**F. Average Uplift**

\[
\text{Average Uplift} =
\text{Pressure Average}
-
\text{Overall Average}
\]

Positive uplift indicates that the player's pressure metric is higher than their overall baseline.

#### Batter Eligibility

A batter is included in the ranking only if they have:

- At least **30 pressure balls**.
- Played in at least **5 matches** containing pressure deliveries.

These thresholds reduce the effect of very small samples. They are model settings and should be reported and tested.

#### Standardisation

The four batter metrics have different units. Therefore, each metric is converted into a z-score:

\[
Z = \frac{x-\mu}{\sigma}
\]

Where:

- \(x\) = player's metric.
- \(\mu\) = mean metric among eligible batters.
- \(\sigma\) = standard deviation among eligible batters.

If a metric has zero standard deviation, its z-score is set to zero.

#### Batter Weighted Score

The model assigns the following weights:

| Metric | Weight |
|---|---:|
| Pressure strike-rate z-score | 45% |
| Pressure average z-score | 25% |
| Strike-rate uplift z-score | 20% |
| Average uplift z-score | 10% |

\[
\begin{aligned}
\text{Raw Batter Score} ={}&
0.45Z_{\text{Pressure SR}}\\
&+0.25Z_{\text{Pressure Average}}\\
&+0.20Z_{\text{SR Uplift}}\\
&+0.10Z_{\text{Average Uplift}}
\end{aligned}
\]

#### Confidence Adjustment

To reduce the influence of players with relatively small pressure samples:

\[
\text{Batter Confidence} =
\frac{B}{B+40}
\times
\frac{M}{M+3}
\]

Where:

- \(B\) = pressure balls faced.
- \(M\) = matches containing pressure deliveries.

\[
\text{Final Batter Rating} =
\text{Raw Batter Score}
\times
\text{Batter Confidence}
\]

Eligible batters are sorted by final rating in descending order. Pressure balls are used as a secondary sorting key.

### 3.4 Bowler Performance Ranking – `bowler_ranking.ipynb`

Bowler performance is evaluated using pressure economy, wicket rate and improvement compared with the bowler's overall baseline.

#### Metrics

**A. Pressure Runs Conceded**

Runs charged to the bowler on pressure deliveries. Byes, leg-byes and penalty runs should not be charged to the bowler. Wides and no-balls are treated according to the source's extras definitions.

**B. Legal Balls**

Legal deliveries bowled in pressure situations. Wides and no-balls are excluded.

**C. Pressure Economy Rate**

\[
\text{Pressure Economy} =
\frac{\text{Pressure Runs Conceded}}
{\text{Pressure Legal Balls}}\times 6
\]

A lower economy rate is better.

**D. Pressure Wickets**

Wickets credited to the bowler in pressure situations. Run-outs, retired hurt, retired out and obstructing-the-field dismissals are not credited to the bowler. Other dismissal types should be checked against the source's dismissal coding.

**E. Pressure Wicket Rate**

\[
\text{Pressure Wicket Rate} =
\frac{\text{Pressure Wickets}}
{\text{Pressure Legal Balls}}\times 6
\]

**F. Economy Improvement**

\[
\text{Economy Improvement} =
\text{Overall Economy}
-
\text{Pressure Economy}
\]

A positive value indicates a lower economy rate under pressure.

**G. Wicket-Rate Uplift**

\[
\text{Wicket-Rate Uplift} =
\text{Pressure Wicket Rate}
-
\text{Overall Wicket Rate}
\]

A positive value indicates a higher wicket rate under pressure.

#### Bowler Eligibility

A bowler is included only if they have:

- At least **60 pressure legal balls**.
- Bowled in at least **5 matches** containing pressure deliveries.

#### Standardisation

Each metric is converted to a z-score among eligible bowlers. Since lower economy is better, the pressure economy z-score is inverted:

\[
Z_{\text{Economy}} =
-\frac{x-\mu}{\sigma}
\]

The other metrics use the normal z-score formula.

#### Bowler Weighted Score

| Metric | Weight |
|---|---:|
| Inverted pressure economy z-score | 40% |
| Pressure wicket-rate z-score | 25% |
| Economy improvement z-score | 20% |
| Wicket-rate uplift z-score | 15% |

\[
\begin{aligned}
\text{Raw Bowler Score} ={}&
0.40Z_{\text{Pressure Economy (inverted)}}\\
&+0.25Z_{\text{Pressure Wicket Rate}}\\
&+0.20Z_{\text{Economy Improvement}}\\
&+0.15Z_{\text{Wicket-Rate Uplift}}
\end{aligned}
\]

#### Confidence Adjustment

\[
\text{Bowler Confidence} =
\frac{B}{B+60}
\times
\frac{M}{M+3}
\]

Where:

- \(B\) = pressure legal balls bowled.
- \(M\) = matches containing pressure deliveries.

\[
\text{Final Bowler Rating} =
\text{Raw Bowler Score}
\times
\text{Bowler Confidence}
\]

Eligible bowlers are sorted by final rating in descending order. Pressure legal balls are used as a secondary sorting key.

## 4. Final Outputs

The project produces two ordered player lists:

1. **Top 10 Batters** – ranked by final batter rating.
2. **Top 10 Bowlers** – ranked by final bowler rating.

The ranking CSV files should include, at minimum:

### Batter output

- Rank
- Player ID and name
- Pressure runs
- Pressure balls
- Pressure matches
- Pressure strike rate
- Pressure average
- Overall strike rate and average
- Strike-rate and average uplift
- Final rating

### Bowler output

- Rank
- Player ID and name
- Pressure wickets
- Pressure legal balls
- Pressure matches
- Pressure economy
- Pressure wicket rate
- Overall economy and wicket rate
- Economy improvement
- Wicket-rate uplift
- Final rating

The actual Top 10 names and values must be generated from the supplied datasets. They should not be manually invented or filled with illustrative players.

## 5. Data Validation and Reliability

Before using the rankings in a submission, perform the following checks:

1. Confirm that all six input files were loaded.
2. Check duplicate records and missing match/player identifiers.
3. Confirm that innings and delivery ordering are correct.
4. Verify that legal balls exclude wides and no-balls.
5. Verify batter-ball and batter-run calculations.
6. Verify bowler-charged runs and bowler-credited wickets.
7. Check revised targets and revised innings limits.
8. Confirm that pressure state is calculated before each delivery.
9. Check that each ranked player meets the minimum sample requirements.
10. Manually inspect a sample of pressure deliveries and player calculations.
11. Test alternative pressure thresholds and compare how much the rankings change.

If a required column is absent or its meaning is uncertain, the program should stop with a clear error rather than silently use an incorrect substitute.

## 6. Limitations

- The pressure definition is a project-specific rule and may not capture every type of match pressure.
- The model uses the supplied competition sample; different competitions may have different playing conditions and standards.
- Player roles, batting positions, bowling phases, venue, opposition strength and match context are not fully adjusted for.
- A player's overall baseline includes pressure deliveries, so the pressure and baseline samples overlap.
- Z-scores are relative to the eligible player pool. Rankings can change when the sample or eligibility thresholds change.
- Confidence adjustment reduces the influence of small samples but does not eliminate uncertainty.
- Incorrect or inconsistent source data can affect pressure classification and player scores.
- The rankings describe past performance in the selected data. They are not forecasts of future performance, player availability or team fit.

## 7. Reproducibility

The project is implemented in Python using Pandas and NumPy.

Install the dependencies:

```bash
pip install pandas numpy
```


```

Run each command from the main `SupeRR_Selector` project directory.

The input CSV files should remain in the `dataset/` folder. Processed files and ranking outputs should be saved in their respective folders.

## 8. Conclusion

The SupeRR Selector methodology uses delivery-level cricket data to identify high-pressure match situations and evaluate player performance in those situations.

Batters are evaluated using pressure strike rate, batting average and improvement over their overall baselines. Bowlers are evaluated using pressure economy, wicket rate and improvement over their overall baselines. Minimum sample requirements, standardised metrics and confidence adjustments are used to make the rankings more consistent.

The final Top 10 batter and bowler lists should be submitted only after validating the source-data schema, cricket calculations, pressure definition and ranking outputs. This makes the analysis more transparent, explainable and reproducible.
