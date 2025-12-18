# NBA Next-Game Points Prediction

**Authors:** Jae Huang (jh9004), Hanzhe Jiang (hj2614)

**Course:** Data Bootcamp – Final Project

**Data Source:** balldontlie API (Endpoint: stats)

**Target:** next_pts (points scored in the player’s next game)

---

## Repository Structure

This repository separates code from analysis by using two notebooks: one focused on data collection and exploratory analysis, and one focused on modeling. The write-up (this README) summarizes the full pipeline and results, referencing figures and visualizations generated in the notebooks.

Notebooks:

*01_eda.ipynb* – API collection, basic cleaning, and exploratory checks; exports a reproducible raw snapshot.

*02_modeling.ipynb* – feature engineering, time-based split, XGBoost model training, and evaluation diagnostics.

---

## Introduction

This project implements a machine learning pipeline to predict how many points an NBA player will score in their next game using recent player-game box score data collected from the balldontlie API. The workflow is separated into two stages: exploratory data analysis (EDA) for data validation and inspection, and modeling for feature engineering, training, and evaluation. The predictive task is framed as a regression problem: given a player’s recent box score history and contextual indicators (minutes, shooting efficiency trends, rest days, opponent tendencies), predict points scored in the player’s next game. Model performance is evaluated using RMSE on a held-out, time-based test set, reported in points.

---

## Data Description

The dataset is sourced from the balldontlie API and consists of player-game stat lines across all NBA teams during the selected date range. The EDA notebook retrieves the data from the API, normalizes JSON responses into a tabular format, and applies basic cleaning to ensure consistency. In regards to the data, it includes a mix of numeric variables (points, minutes, shooting statistics, rebounds, assists) and identifiers (player IDs, game IDs, team IDs) that support time-aware feature construction. Missingness analysis is performed to assess data quality. Most variables are fully populated, while overtime-related fields exhibit high missing rates due to the rarity of overtime games rather than data integrity issues.

We do want to make note of a deliberate design choice, which is the use of a short, recent time window (2025-10-01 to 2025-12-01). Player scoring outcomes exhibit strong recency effects driven by lineup changes, injuries, role adjustments, coaching decisions, and pace variation. Including older games introduces stale information, degrading predictive performance. Therefore, our current model prioritizes recency over long-term historical coverage. The main point we would like to highlight is that at the time this project was a work in progress, the most recent 2025–2026 NBA season had just started in November 2025. As a result, there were limitations in terms of data collection, which may have impacted our model performance (more data over time could have resulted in better predictions). Despite the constraints in terms of available data for the most recent season, for this project we decided to continue with this short, recent time window, as using older historical data would be detrimental in the NBA setting. Player roles and usage can shift significantly with roster moves, coaching changes, trades, and evolving rotations; injury and return timelines can also abruptly change minutes and shot volume. In addition, team-level systems (pace, offensive style, and matchup strategies) can change across seasons, which reduces the relevance of older games for predicting next-game outcomes. For these reasons, we treat additional data as helpful only when it is recent and contextually comparable, and we expect performance to improve as more current-season observations become available while maintaining a recency-focused training approach.

---

## Exploratory Data Analysis (EDA)

Exploratory analysis is conducted in the 01_eda.ipynb notebook and is limited strictly to data collection, validation, and basic cleaning. No feature engineering or modeling-related transformations are introduced at this stage. The analysis is based on 10,779 player–game observations from the 2025 NBA season, providing a well-structured dataset with heterogeneous variable types.

Based on the results of data quality checks, high overall completeness across most fields was confirmed. Some steps were taken to clean up the data slightly before use, including removing duplicate player–game records and standardizing key fields such as game date and minutes played. Distributional analysis shows that both points scored (pts) and minutes played (min) are strongly right-skewed, with most observations clustered at low values and a small number of high-usage performances forming a long tail. However, we do want to mention that quantile analysis confirms that high-scoring games represent valid but infrequent outcomes rather than anomalies.

Overall, this step just checks that the dataset is reliable and representative of NBA scoring behavior, providing a suitable foundation for feature engineering and modeling. The cleaned data is saved as a raw snapshot for reproducibility, and this notebook intentionally excludes any modeling logic.

<img width="549" height="393" alt="image" src="https://github.com/user-attachments/assets/f2494071-8692-4764-a667-56cf23caccc8" />

*(Visual graph of distribution of points scored per game)*

---

## Models and Methods

### Train/test split strategy

To reflect real-world deployment, the dataset is split chronologically rather than randomly. For instance, earlier games are used for training and later games for testing as this approach prevents future information leakage and preserves temporal structure.

### Feature engineering

The modeling notebook constructs rolling window features to summarize recent player performance without using contemporaneous information from the target game. Rolling means over 3, 5, and 10 games are computed for points, rebounds, assists, field goal percentage, and minutes, with all features shifted to ensure temporal validity.

Additional features include season-level averages and standard deviations to stabilize player baselines, rest-day calculations to capture fatigue and recovery, and opponent-context features. Opponent “allowed” statistics are computed by aggregating opponent outcomes by date and applying shifted rolling averages, enabling the model to account for defensive context without leakage. A basic usage proxy is also constructed and rolled to capture changes in offensive involvement.

<img width="1169" height="547" alt="image" src="https://github.com/user-attachments/assets/4e6d0f69-7cd9-418c-927d-0f8a983db3fd" />

*(Example of Stephan Curry: real points over time plot)*

<img width="1171" height="547" alt="image" src="https://github.com/user-attachments/assets/2daa86c4-11de-4fd6-89f9-352aead505bc" />

*(Example of Giannis Antetokounmpo: rolling points feature plot)*

### Model choice

The final model is an XGBoost regressor with fixed hyperparameters. 

---

## Results and Interpretation

Model performance is evaluated on the held-out test set using RMSE, which represents the typical prediction error in points.

<img width="696" height="695" alt="Screenshot 2025-12-17 at 12 11 01 AM" src="https://github.com/user-attachments/assets/637010e8-6f83-4bcb-8d3e-ee43fa62ea5e" />

*(Predicted points vs actual points scatter plot, x=y reference line)*

Feature importance analysis highlights the dominance of rolling performance metrics and playing-time indicators. These features encode both recent form and opportunity, which are primary drivers of scoring outcomes.

<img width="719" height="455" alt="image" src="https://github.com/user-attachments/assets/e0ff5925-eaec-4082-bc2b-aee9fea53975" />

*(Plot displays top 30 feature importances w/ respective importance scores)*

Error analysis is conducted at both the player and outcome levels. Average error by player identifies individuals who are consistently under- or over-predicted, often due to volatile roles, minute uncertainty, or missing contextual information such as injuries. Plotting error against actual points shows that the model performs best for typical scoring ranges and exhibits higher error for extreme high-scoring performances, which are rare and driven by situational factors not fully captured by box score data.

<img width="853" height="470" alt="image" src="https://github.com/user-attachments/assets/b5199f09-da0e-4fe2-80c8-31e07d7b292a" />

*(Predicted points vs actual points scatterplot)*

<img width="280" height="466" alt="Screenshot 2025-12-17 at 12 14 43 AM" src="https://github.com/user-attachments/assets/c02c88c5-edc2-409c-8416-3e7538a1ce58" />

*(Table of most under-predicted / over-predicted players by player.id)*

A naive baseline such as predicting a player’s recent 5-game average would typically yield a similar or worse RMSE, while XGBoost provides a structured way to incorporate multiple contextual signals beyond simple averaging.

---

## Conclusion and Next Steps

All in all, this project features a complete end-to-end predictive workflow to predict NBA players' next-game points; it includes API data collection, data cleaning, exploratory analysis, feature engineering, time-aware evaluation, and regression-based prediction of next-game scoring. Results indicate that recent performance and playing-time signals are strong predictors, while the most challenging cases involve high-variance players and extreme scoring outcomes.

We do want to highlight that, in terms of limitations, our current model relies on box score–derived statistics and short-term rolling trends. While these may be effective for capturing recent form, they do not fully represent the game-level context that influences scoring. After taking into consideration the other endpoints and signals avalibale on balldontlie API, we believe that there are several that could be implemented and utlized in a future model for more accurate and stable player next-game point predictions. For future improvements, some additional signals that can be incorporated from the API are: betting spreads and totals to proxy expected pace and competitiveness, implied win probabilities to capture game flow and late-game usage, and season-level scoring baselines to stabilize predictions when recent windows are noisy—could further reduce mid-range prediction error and improve overall robustness in future iterations. 

---

## Reproducibility and Setup

To reproduce this project:

* Clone the repository and open the notebooks.
* Add your own API key in the notebooks where `api_key` is defined.
* Run 01_eda.ipynb first to fetch and clean data, then export the raw snapshot CSV.
* Run 02_modeling.ipynb to engineer features, train the model, and generate evaluation figures.

Dependencies: numpy, pandas, requests, matplotlib, seaborn, scikit-learn, xgboost, tqdm
Note on files: The modeling notebook reads raw_data.csv. Ensure that the exported filename and path match what the modeling notebook expects.
