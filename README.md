 Crime pattern analysis and predictive modeling in Los Angeles

A data mining project applying classification, clustering, and association
rule mining to LAPD crime records (2020–2025) to explore temporal,
spatial, and demographic patterns in crime, and to test whether victim
demographics can be predicted from situational crime attributes.

Authors:

- Frederic Bomba Mbida  International Political Economy and Development, Fordham University  fb26@fordham.edu
- Philip Chan  International Political Economy and Development, Fordham University  pchan16@fordham.edu



 Objective

This project asks a simple question: do victim demographics (race, gender)
follow predictable patterns based on other features of a crime  time,
location, weapon, crime type or are they largely independent of them?

The motivation came from public debate in Los Angeles around rising crime
rates and prosecutorial policy (e.g. the recall of District Attorney George
Gascón). Rather than relying on political narratives, this project uses data
mining to check whether crime outcomes follow measurable, structural patterns
(time of day, location, situational context) rather than being driven by
who the victim is.

To do this, we combined:
- Classification : predict victim race/gender from other crime attributes
  (Decision Trees, Random Forest, Neural Networks)
- Clustering : group crimes by shared characteristics to find natural
  structure in the data (K-Means, Hierarchical Clustering, DBSCAN, Birch)
- Association rule mining: find attribute combinations that co-occur
  more often than chance (Apriori)
- Geospatial mapping: visualize where crimes and clusters concentrate
  across Los Angeles (Folium)

Headline finding: classification and association rules were only weakly
predictive of victim demographics (see Results(results-summary) below).
Situational and temporal factors,  time of day, location, weapon, crime
type, explain crime patterns far better than victim identity does. This is
consistent with routine activity theory (Cohen & Felson, 1979) and
environmental criminology (Brantingham & Brantingham, 1993), which argue
that crime opportunity, not victim characteristics, drives when and where
crime happens.


 Repository Contents

| File | Description |
|---|---|
| `Crime pattern in LA.docx` (paper) | Full write-up: background, methodology, results, tables, and discussion. Read this first for context on *why* each analysis was run and how to interpret it. |
| `Crime LA code2.txt` (code) | The full analysis pipeline, from raw CSV to classification, clustering, mapping, and association rules. Heavily commented so it can be run and understood step by step. |
| `Crime_Data_from_2020_to_Present.csv` (not included — see below) | Raw LAPD dataset. Too large to commit to the repo; download separately (see Data)). |

> Note: the paper reports results from Decision Trees, Random Forest, and
> Neural Network classifiers, and from K-Means and Hierarchical clustering.
> The accompanying code file implements the Random Forest classification and
> K-Means/DBSCAN/Birch clustering pipelines in full; the Decision Tree and
> Neural Network experiments referenced in the paper were run separately and
> are not included in this version of the script.

---

 Data

Source: LAPD Open Data Portal, "Crime Data from 2020 to Present"
https://data.lacity.org

Each row represents one reported crime incident, with fields for the date,
time, location (LAT/LON), area, crime type, weapon used, and victim
demographics (age, sex, descent).

To replicate this project:
1. Download `Crime_Data_from_2020_to_Present.csv` from the LAPD Open Data
   Portal.
2. Place it in the same directory as the script (or update the `path`
   variable at the top of `crime_analysis_annotated.py`).
3. The script only loads the first 100,000 rows (`nrows=100000`) to keep
   runtime and memory usage manageable — increase or remove this limit if
   you want to run on the full dataset.



 Process / Methodology

The pipeline follows these stages, in order:

 1. Data Cleaning
- Drop identifier and free-text columns not needed for analysis (case
  numbers, status codes, exact addresses, etc.).
- Combine `DATE OCC` and `TIME OCC` into a single datetime column; extract
  hour, weekday, and a weekend flag.
- Bucket hours into `morning` / `afternoon` / `evening` / `night`.
- Decode single-letter victim descent and sex codes into readable labels
  (e.g. `H` → `Hispanic/Latin/Mexican`, `F` → `Female`).
- Drop rows with missing/invalid coordinates (0,0 = no GPS fix).

 2. Exploratory Analysis
- Crime counts by time of day, weekday, and month (seasonality).
- Victim descent and gender distributions.
- Crime volume by hour, broken down by victim descent and gender.

These plots motivated the paper's temporal-pattern findings: crime peaks
around midday and again in the evening, and follows a seasonal curve tied to
weather and daylight.

 3. Classification (Predicting Victim Descent)
- Features: hour, victim age, latitude/longitude (numeric); area, premise,
  crime type, weapon, weekday, time-of-day bucket (categorical).
- Target: victim descent (race/ethnicity).
- Preprocessing: `StandardScaler` for numeric features, `OneHotEncoder`
  for categorical features, combined via `ColumnTransformer`.
- Class imbalance: victim descent categories are heavily skewed (White
  and Hispanic victims vastly outnumber Pacific Islander, Cambodian, etc.).
  SMOTE (Synthetic Minority Oversampling) was applied inside an
  `imblearn` pipeline so oversampling only ever touches the training data,
  never the test set, avoiding data leakage.
- Model: `RandomForestClassifier` (50 trees, max depth 10) — a lighter
  configuration chosen to control runtime and memory on the 100k-row sample.
- Evaluation: classification report (precision/recall/F1 per class) and
  a confusion matrix heatmap.

 4. Clustering
Several clustering runs were performed with increasingly rich feature sets,
each visualized on a Folium map with sampled points and a legend:
- K-Means on location + hour + age
- K-Means on victim descent + sex only (with PCA for 2D visualization)
- K-Means on a broader mix of age, sex, descent, time, and location
- K-Means on the full feature set including area, crime type, premise,
  and weapon
- Birch as an alternative to K-Means on the same full feature set
- DBSCAN (density-based) on location, age, hour, sex, and descent — this
  method labels sparse points as noise (`-1`) rather than forcing every
  point into a cluster

For each K-Means/Birch run, cluster "profiles" were computed (average age,
dominant gender/race, peak time) to make clusters interpretable rather than
just numbered groups.

 5. Association Rule Mining
- Selected categorical attributes (crime type, victim sex/descent, time of
  day, weapon, premise) were one-hot encoded into a "basket" format.
- Apriori found frequent attribute combinations (`min_support=0.01`).
- Association rules were derived from those itemsets, filtered to
  `lift ≥ 1.2` (meaning the combination occurs at least 20% more often than
  if the attributes were independent).

 6. Geospatial Visualization
Folium was used throughout to plot crime locations and cluster assignments
on an interactive map of Los Angeles, with `MarkerCluster` for performance
and custom HTML legends for interpretability.

---

 Results Summary

| Approach | Outcome |
|---|---|
| Random Forest (victim descent/gender) | Moderate accuracy (~0.68–0.74 weighted F1), but poor performance on minority classes even after SMOTE rebalancing |
| Neural Network | Best overall accuracy (~0.64), but still weak on minority classes; less interpretable |
| K-Means clustering | Produced some coherent, interpretable clusters (e.g. "Hispanic males, robbery, downtown, evening") but others were scattered and not clearly separable |
| Hierarchical clustering | Points were scattered across clusters — did not produce clear, interpretable groupings |
| Association rule mining | Found plausible rules (e.g. vehicle theft ~ night + parking lots) but support was low (<0.1), limiting reliability |

Overall conclusion: victim demographics are difficult to predict from
situational crime attributes. Temporal and environmental context (time,
location, weapon, premise) is a much stronger predictor of crime type than
of who the victim is, supporting routine activity theory over narratives
that attribute crime patterns to demographic targeting or prosecutorial
policy alone.



Limitations

- Only the first 100,000 rows of the dataset were used, for performance
  reasons — results may shift with the full dataset.
- Class imbalance in victim descent is severe; SMOTE mitigates but does not
  fully solve this, especially for very small classes (e.g. Cambodian,
  Laotian, Guamanian).
- Association rule support values were generally low, limiting confidence in
  the mined rules.
- Clustering results (especially hierarchical clustering) were sensitive to
  feature scaling and choice of `k`, and were not tuned exhaustively (e.g. no
  elbow-method or silhouette analysis was run to select the optimal number
  of clusters).



 How to Run

```bash
pip install pandas numpy matplotlib seaborn scikit-learn imbalanced-learn mlxtend folium

python crime_analysis_annotated.py
```

Or open the script in Jupyter/Colab and run cell by cell — it was originally
written and tested in a notebook environment.

---

 References

See the full reference list in the paper. Key theoretical grounding:
- Cohen, L. E., & Felson, M. (1979). *Social change and crime rate trends: A routine activity approach.* American Sociological Review.
- Sampson, R. J., & Groves, W. B. (1989). *Community structure and crime: Testing social-disorganization theory.* American Journal of Sociology.
- Brantingham, P. J., & Brantingham, P. L. (1993). *Crime Pattern Theory.* Springer.

Data source: Los Angeles Police Department Open Data Portal — https://data.lacity.org
