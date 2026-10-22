# AI Conversation Transcript: Assignment

This transcript contains all 11 exchanges, preserving the questions, AI responses, explanations, and Python code.

---

**Question**

I'm plotting maintenance records per machine, but the plot runs too long. Remind me how I would make it display vertically?

```python
counts = maintenance_df['machineID'].value_counts().sort_index().reset_index()
counts.columns = ['machineID', 'count']
plt.figure(figsize=(20, 5))
plt.bar(counts['machineID'], counts['count'])
plt.xticks(counts['machineID'], rotation=90, fontsize=10)
plt.yticks(fontsize=12)
plt.title('Maintenance Records per Machine', fontsize=16)
plt.xlabel('Machine ID', fontsize=14)
plt.ylabel('Count', fontsize=14)
plt.tight_layout()
plt.show()
```

**Response**

Use `plt.barh`, swap the axes and make the figure tall:

```python
counts = maintenance_df['machineID'].value_counts().sort_index().reset_index()
counts.columns = ['machineID', 'count']
plt.figure(figsize=(6, 20))
plt.barh(counts['machineID'], counts['count'])
plt.yticks(counts['machineID'], fontsize=8)
plt.gca().invert_yaxis()  # machine 1 at top
plt.title('Maintenance Records per Machine', fontsize=16)
plt.xlabel('Count', fontsize=14)
plt.ylabel('Machine ID', fontsize=14)
plt.tight_layout()
plt.show()
```

---

**Question**

I am trying to calculate the number of days between dates. Here is what I have:

```python
m = maintenance_df.sort_values(["machineID", "comp", "datetime"])
m["days_since_prev"] = m.groupby(["machineID", "comp"])["datetime"].diff()
```

But how would I get just number of days?

**Response**

Add `.dt.days` to the end:

```python
m["days_since_prev"] = m.groupby(["machineID", "comp"])["datetime"].diff().dt.days
```

`.dt.days` keeps only whole days. If you want fractional days, use `.dt.total_seconds() / 86400` instead.

---

**Question**

I tried to group up any failures for a given machine that occur at the same timestamp. Why doesn't this print out as a normal-looking DataFrame?

```python
events_df = failures_df.groupby(["machineID", "datetime"])['failure'].apply(sorted)
```

**Response**

Selecting one column (`['failure']`) and then calling `.apply` gives you a Series with a MultiIndex (`machineID`, `datetime`), not a DataFrame. Add `.reset_index()` to get the grouping keys back as columns:

```python
events_df = failures_df.groupby(["machineID", "datetime"])['failure'].apply(sorted).reset_index()
```

Each row's `failure` value will be a list such as `['comp1', 'comp2']`.

---

**Question**

Remind me how to plot multiple subplots for the sensor data in `telemetry_m1_df` from January 2015.

**Response**

```python
telemetry_m1_df.loc["2015-01", sensors].plot(subplots=True, figsize=(12, 8), style='.')
```

This picks the January 2015 rows for the sensor columns and plots each sensor in its own subplot as points.

---

**Question**

I want to create a column to my DataFrame called `type` which contains either `repair` or `scheduled` based on whether the `failure` column has a value or is NaN. A value means repair and NaN means scheduled. Please explain how I would do that.

**Response**

You can do this in one line by checking whether `failure` is filled in:

```python
maintenance_failures_merged_df["type"] = maintenance_failures_merged_df["failure"].notna().map(
    {True: "repair", False: "scheduled"}
)
```

`.notna()` gives you `True` wherever there's a failure value and `False` where it's NaN. Then `.map()` swaps those for labels: `True` becomes `"repair"` and `False` becomes `"scheduled"`. So any maintenance record tied to a failure gets tagged as a repair, and the rest count as scheduled.

---

**Question**

Take a look at `failures_with_error_count_df`. I want to get the percentages instead of the count. How would I do that?

**Response**

To get percentages, divide each row by that component's total number of failures:

```python
failures_with_error_pct_df = failures_with_error_count_df.div(failures_df["failure"].value_counts(), axis=0).mul(100).round(1)
```

`value_counts()` gives the total failures per component. `.div(..., axis=0)` divides each row by its component's total, and `.mul(100).round(1)` turns that into a percentage. Each cell now reads as "X% of compN failures had this error".

---

**Question**

Take a look at `plot_telemetry_data_around_failure()`. I want to now create a function that would plot the average across all failures for a given component. Help me create this function.

**Response**

The idea is to take the same ±days window as `plot_telemetry_data_around_failure()`, but for every failure of that component. Re-time each window as "hours from failure" so they all line up, then average:

```python
def plot_average_around_failure(comp, days=5):
    comp_failures_df = failures_df[failures_df["failure"] == comp]
    windows = []

    for _, failure in comp_failures_df.iterrows():
        start = failure["datetime"] - pd.Timedelta(days=days)
        end = failure["datetime"] + pd.Timedelta(days=days)

        window_df = telemetry_df[
            (telemetry_df["machineID"] == failure["machineID"]) &
            (telemetry_df["datetime"] >= start) &
            (telemetry_df["datetime"] <= end)
        ].copy()

        window_df["hours_from_failure"] = (window_df["datetime"] - failure["datetime"]).dt.total_seconds() / 3600
        windows.append(window_df)

    all_windows_df = pd.concat(windows)
    average_df = all_windows_df.groupby("hours_from_failure")[sensors].mean()

    axes = average_df.plot(
        subplots=True,
        figsize=(12, 8),
        title=f"Average telemetry around {comp} failures"
    )
    for ax in axes:
        ax.axvline(0, color="black", linestyle="--")
    plt.show()
```

`hours_from_failure` is the key piece: it puts every failure at 0, so the `groupby(...).mean()` averages the same relative hour across all failures. The dashed line marks the failure point. Averaging also cancels out most of the noise, so you don't need the rolling mean here.

---

**Question**

I created the lift function. Help me create a function for calculating precision. In the assignment, it is described as:

> Precision: of the alerts it raises (each time it switches from off to on), how often an event follows within H + W.

**Response**

The trick is to count each alert once, at the hour it switches from off to on, and then check how many of those starts land inside a warning window:

```python
def calc_precision(alert_col):
    was_on_last_hour = grid_df.groupby("machineID")[alert_col].shift(1, fill_value=False)
    alert_starts_df = grid_df[grid_df[alert_col] & ~was_on_last_hour]
    return alert_starts_df["in_window"].mean()
```

- `shift(1)` within each machine gives the alert state from the previous hour. Grouping by `machineID` stops one machine's last hour from leaking into the next machine's first hour.
- `alert_col & ~was_on_last_hour` keeps only the off→on hours, so a 10-hour alert counts as one alert, not ten.
- `in_window` is `True` when that hour falls in the H + W window before a failure, which means a failure follows within H + W. Taking `.mean()` of a boolean column gives the fraction of alert starts that were followed by a failure, and that's your precision.

---

**Question**

Okay, so I've got `grid_df` with every machine-hour. Now I want to flag the hours that fall in the warning window before each failure, so between H + W and H hours before it. Can I also keep track of which failure each hour belongs to? I'll need that later for coverage.

**Response**

You can loop over the events and label the matching hours in `grid_df` with each failure's ID:

```python
grid_df["window_id"] = None

for failure_id, failure in events_df.iterrows():
    window_start = failure["datetime"] - pd.Timedelta(hours=H + W)
    window_end = failure["datetime"] - pd.Timedelta(hours=H)

    grid_df.loc[
        (grid_df["machineID"] == failure["machineID"]) &
        (grid_df["datetime"] >= window_start) &
        (grid_df["datetime"] < window_end),
        "window_id"
    ] = failure_id

grid_df["in_window"] = grid_df["window_id"].notna()
```

For each failure, this finds that machine's hours between H + W and H hours before the failure and stamps them with the failure's ID. That's what lets you track which failure each hour belongs to. `in_window` then just checks whether an hour got an ID, which gives you a simple True/False flag for lift and precision. For coverage, you can count `window_id.nunique()` on the hours where an alert fired.

---

**Question**

My volt alert keeps flickering on and off, resulting in way too many alerts and also messing up my precision. How could I keep the alert on or as one for a span of time so all the flickering aren't considered separate alerts?

**Response**

You can keep the alert on for 12 hours after it last fired. That way, short gaps between flickers get filled in and the whole burst counts as one alert:

```python
grid_df["volt_alert_steady"] = (
    grid_df.groupby("machineID")["volt_alert"]
    .transform(lambda x: x.rolling(12, min_periods=1).max())
    .astype(bool)
)
```

The rolling `max` over 12 hours asks "did the alert fire at any point in the last 12 hours?" If it did, the steady alert stays on. Any off-gap shorter than 12 hours gets bridged, so `calc_precision` sees one off→on switch instead of a bunch of them. Grouping by `machineID` keeps it from carrying over between machines. `min_periods=1` lets it work from the first few rows instead of giving NaN.

---

**Question**

Ok so in `02_precursors` I worked through comp1 and built 4 precursors, then compared them with lift, precision and coverage. Can you first read through `02_precursors` thoroughly, and then help me repeat that same process for all 4 components in a new notebook, `03_all_component_precursors`? Each component has its own error(s) and sensor from the EDA in `01_data_exploration`. Note that for all sensor values, the abnormal values rise except for rotate which drops rotate drops. I'll write up the findings myself. Ask me any relevant questions as we build this out.

**Response (output note)**

Most of the Python notebook `03_all_component_precursors` was created in response to this question along with further discussion. The response primarily consisted of notebook creation rather than a conventional written answer.

---
