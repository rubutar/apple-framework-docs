# Reading & Writing HealthKit Data

This document focuses on **how HealthKit data is actually read and written**, and why apps often receive empty results even when permissions are granted.

This is not a tutorial — it explains **system behavior and decision points**.

---

## What This File Solves

- “Why does my query return no data?”
- “Why does saving succeed but nothing appears in Health?”
- “When should I use samples vs workouts?”
- “Why does HealthKit behave differently on Watch?”

---

## 1. Reading Data from HealthKit

Reading data in HealthKit is done through **queries**, not direct access.

Key rules:
- Authorization does **not** guarantee data exists
- Queries are **time-bound**
- Results depend on **source priority** and **availability**

---

### 1.1 Common Query Types

- **Sample Queries**  
  Used to fetch raw samples (heart rate, HRV, sleep, mindfulness).

- **Statistics Queries**  
  Used to aggregate values (average, sum, min/max).

- **Anchored Queries**  
  Used for incremental updates over time.

Most empty-result issues come from **sample queries with incorrect predicates**.

---

### 1.2 Why Queries Return Empty Results

Empty results usually mean **one of these is true**:

- No data exists in the requested time range
- The data type is not available on the device
- The app is not an authorized source
- The predicate is too restrictive
- The user has no relevant data yet

HealthKit does **not** throw errors for these cases.

---

### Swift Reference — Basic Sample Query

```swift
let predicate = HKQuery.predicateForSamples(
    withStart: startDate,
    end: endDate,
    options: []
)

let query = HKSampleQuery(
    sampleType: quantityType,
    predicate: predicate,
    limit: HKObjectQueryNoLimit,
    sortDescriptors: nil
) { _, samples, error in
    // samples may be empty even when authorization is granted
}
```

## 2. Writing Data to HealthKit

Writing data means saving samples into the HealthKit store.

Important:

- Apps do not overwrite system data
- Data may not appear immediately
- Visibility depends on data type and source priority

### 2.1 Writing Quantity Samples

Quantity samples represent numeric values with units.

Rules:
- Units must match the data type
- Timestamps must be realistic
- Metadata is optional but useful

Swift Reference — Saving a Quantity Sample

```
let sample = HKCategorySample(
    type: categoryType,
    value: categoryValue,
    start: startDate,
    end: endDate
)

healthStore.save(sample, completion: { _, _ in })
```

### 4. Using Workouts as Data Containers

Workouts are not just fitness activities.

They:
- Enable background execution on Apple Watch
- Group related samples
- Improve system understanding of context

Common mistake:
Using category samples when a workout is required (or vice versa)

### 5. Why Saved Data Doesn’t Show Up in Health App
Possible reasons:
- Health app filters hide the data
- Source priority places your app below system sources
- Data type is not surfaced prominently
- The time range is outside the visible window

HealthKit does not guarantee UI visibility for third-party data.

### 6. Watch vs iPhone Behavior

Key differences:
- Authorization always happens on iPhone
- Watch relies on iPhone permissions
- Background execution requires workouts or system triggers

If data is written on Watch:
- It may sync later
- It may not appear immediately
- It may be merged with iPhone data





## HealthKit API Summary — Beginner

These APIs cover **core usage** and are required for most HealthKit apps.

| Area | API / Type | Purpose | Notes |
|----|-----------|--------|------|
| Core | `HKHealthStore` | Entry point to HealthKit | Required for all HealthKit access |
| Authorization | `requestAuthorization` | Request read/write permissions | Must be called before queries or saves |
| Authorization | `authorizationStatus(for:)` | Check permission state | Does not prompt user |
| Data Types | `HKQuantityType` | Numeric health data | Heart rate, HRV, steps |
| Data Types | `HKCategoryType` | State-based data | Sleep, mindfulness |
| Data Types | `HKWorkoutType` | Workout records | Fitness or mind & body |
| Samples | `HKQuantitySample` | Stored numeric data | Requires unit |
| Samples | `HKCategorySample` | Stored categorical data | Requires value + time range |
| Samples | `HKWorkout` | Activity container | Groups related samples |
| Queries | `HKSampleQuery` | Read raw samples | Most common query |
| Writing | `save(_:completion:)` | Write data to HealthKit | Success ≠ immediate visibility |
| Units | `HKUnit` | Measurement units | Mandatory for quantity samples |

---

## HealthKit API Summary — Advanced

These APIs are used for **background behavior**, **Apple Watch**, and **complex data flows**.

| Area | API / Type | Purpose | When to Use |
|----|-----------|--------|------------|
| Queries | `HKStatisticsQuery` | Aggregate data | Averages, sums, min/max |
| Queries | `HKAnchoredObjectQuery` | Incremental updates | Syncing new data over time |
| Queries | `HKObserverQuery` | Background notifications | React to HealthKit changes |
| Background | `enableBackgroundDelivery` | Receive background updates | Long-running monitoring |
| Watch | `HKWorkoutSession` | Manage Watch workout lifecycle | Enable background execution |
| Watch | `HKLiveWorkoutBuilder` | Collect live Watch data | Real-time metrics |
| Metadata | `HKMetadataKey*` | Add context to samples | Device, algorithm, user-entered |
| Sources | `HKSource` | Identify data origin | Debugging & prioritization |
| Deletion | `delete(_:completion:)` | Remove app-created data | Corrections, cleanup |
| Characteristics | `HKCharacteristicType` | Static user attributes | Read-only user info |

---

## How to Read These Tables

- **Beginner APIs** → Required for most HealthKit apps  
- **Advanced APIs** → Needed for Watch apps, background delivery, and analytics  
- You do **not** need all APIs to ship a valid HealthKit app

Start simple. Add advanced APIs **only when the system behavior requires it**.
