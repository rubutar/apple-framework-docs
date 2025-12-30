# HealthKit

## 1. Overview

HealthKit is Apple’s framework for accessing and storing health and fitness data in a centralized, secure location on the user’s device. It allows apps to read and write health-related data with explicit user permission, while maintaining strong privacy guarantees.

HealthKit itself does not collect data. Instead, it acts as a **data broker** between:
- Data sources (Apple Watch, iPhone sensors, third-party apps)
- Data consumers (apps with approved permissions)
- The system Health app, which acts as the user-facing control center

---

### 1.1 What HealthKit Is

HealthKit is:
- A **secure data store** for health and fitness information
- A **permission-based access layer** controlled by the user
- A **unified data model** that normalizes data from many sources
- A **long-term record**, not a real-time streaming framework

Apps interact with HealthKit through structured queries and explicit save operations, not through continuous live feeds.

---

### 1.2 What HealthKit Is NOT

HealthKit is **not**:
- A sensor API (you do not read heart rate directly from HealthKit)
- A real-time monitoring framework
- A background execution guarantee
- A replacement for Core Motion or Core Bluetooth

If your app needs **live heart rate**, **live motion**, or **continuous background execution**, those are handled by other frameworks (often in combination with HealthKit).

---

### 1.3 Data Ownership and Privacy Model

A core principle of HealthKit is that **the user owns their health data**.

Key privacy rules:
- Apps must explicitly request permission for each data type
- Read and write permissions are granted separately
- Users can revoke permissions at any time
- Apps never see data they are not authorized to access
- Health data is encrypted and protected by the system

The Health app is the **single source of truth** for managing permissions and reviewing collected data.

> ⚠️ **Important**  
> Even if authorization is granted, HealthKit may still return no data due to availability, prioritization rules, or data source constraints.

---

### 1.4 Platform Roles (High-Level)

- **iPhone**
  - Owns HealthKit authorization
  - Stores the primary HealthKit database
  - Hosts the Health app UI

- **Apple Watch**
  - Acts primarily as a data source
  - Collects sensor data (heart rate, HRV, workouts)
  - Syncs data to the iPhone via the system

Apps must respect this separation when designing their HealthKit architecture.

---

### 1.5 When to Use HealthKit

Use HealthKit when your app:
- Needs access to historical health or fitness data
- Contributes meaningful health data to the system
- Integrates with Apple Watch health metrics
- Respects user privacy and system constraints

Do **not** use HealthKit for:
- Live sensor visualization
- Continuous background tracking without workouts
- Data that is not health-related

## 2. HealthKit Architecture

Understanding HealthKit’s architecture is critical to avoiding common design mistakes. Many issues developers encounter—missing data, background failures, or Watch-only assumptions—come from misunderstanding how responsibilities are split across the system.

HealthKit is not a single database owned by your app. It is a **system-level service** that coordinates data between devices, apps, and the user.

---

### 2.1 System Components

HealthKit architecture consists of three main components:

1. **HealthKit Framework**
   - Provides APIs for reading and writing health data
   - Enforces authorization and privacy rules
   - Normalizes data into a unified model

2. **Health App**
   - User-facing interface for health data
   - Manages permissions and data visibility
   - Displays aggregated data from all sources

3. **Data Sources**
   - Apple Watch (primary source for many metrics)
   - iPhone sensors
   - Third-party apps and devices

Apps never communicate directly with other apps’ health data. All access is mediated by HealthKit.

---

### 2.2 iPhone as the Authority

The **iPhone is the authoritative device** for HealthKit.

Responsibilities of the iPhone:
- Stores the primary HealthKit database
- Handles all HealthKit authorization requests
- Enforces privacy and permission changes
- Hosts the Health app UI

Even when data is collected on Apple Watch, it is ultimately synced to and governed by the iPhone.

> ⚠️ **Important**  
> HealthKit authorization must always be requested on the iPhone. Watch-only apps cannot initiate authorization.

---

### 2.3 Apple Watch as a Data Source

The Apple Watch’s role is primarily **data collection**, not data ownership.

Watch responsibilities:
- Collects sensor data (heart rate, HRV, motion)
- Runs workout and mindfulness sessions
- Temporarily stores samples during activity
- Syncs data to the iPhone through the system

The Watch does not:
- Own HealthKit permissions
- Guarantee immediate data availability
- Allow unrestricted background execution

Data collected on the Watch may appear in HealthKit **minutes or hours later**, depending on system conditions.

---

### 2.4 Data Flow Overview

A simplified data flow looks like this:

1. User grants permission on iPhone
2. Apple Watch collects sensor data
3. Data is saved to HealthKit
4. HealthKit syncs data to iPhone
5. Apps query HealthKit on iPhone
6. Health app presents aggregated results

This asynchronous flow explains why querying “right after an activity” often returns no data.

---

### 2.5 Data Source Prioritization

When multiple sources provide the same data type, HealthKit applies **source prioritization**.

Examples:
- Apple Watch heart rate is usually prioritized over third-party sensors
- Manually entered data may override sensor data
- User-defined source order affects query results

Apps cannot force their data to be the preferred source.

> 🧠 **Insight**  
> If your app writes data that appears “ignored,” it may be lower priority than existing sources rather than rejected.

---

### 2.6 Implications for App Design

Because of this architecture:
- Always design for **eventual consistency**
- Avoid assumptions about immediate data availability
- Separate data collection from data visualization
- Treat HealthKit as a long-term record, not a live feed

Apps that respect these constraints are more reliable and more likely to pass App Review.

---

### 2.7 Common Architectural Misconceptions

- “My Watch app can manage HealthKit permissions” → ❌
- “Authorization success means data exists” → ❌
- “HealthKit streams live sensor data” → ❌
- “Background delivery guarantees execution” → ❌

These misconceptions lead directly to unstable apps and confusing user experiences.


## 3. Capabilities, Entitlements, and Setup

Before an app can interact with HealthKit, it must be correctly configured at the project level. HealthKit APIs will compile and run even when setup is incomplete, but data access depends entirely on proper capabilities and entitlements.

This section describes the required setup and how it applies across iOS and watchOS.

---

### 3.1 Enabling the HealthKit Capability

To use HealthKit, the HealthKit capability must be enabled in the app’s target settings.

Enabling the capability:
- Adds the HealthKit entitlement to the app
- Allows the app to request health data permissions
- Is required for both iOS and watchOS targets that access HealthKit

HealthKit must be enabled separately for:
- The iOS app target
- The watchOS app or extension target (if applicable)

---

### 3.2 Required Entitlements

When the HealthKit capability is enabled, Xcode adds the appropriate entitlements automatically.

Key points:
- Entitlements are validated at runtime
- Missing or mismatched entitlements prevent HealthKit access
- The system does not always surface explicit errors for entitlement issues

For apps with both iPhone and Apple Watch components, entitlements must be present on each target that interacts with HealthKit APIs.

---

### 3.3 Info.plist Privacy Descriptions

Apps must include usage descriptions in `Info.plist` to explain why health data is accessed.

Common keys include:
- `NSHealthShareUsageDescription`
- `NSHealthUpdateUsageDescription`

These strings are presented to the user during authorization and should clearly describe:
- What data is accessed
- Why the app needs it
- How the data benefits the user

Authorization requests may fail or be denied if these descriptions are missing or unclear.

---

### 3.4 Device and Account Requirements

HealthKit access depends on system availability.

Requirements include:
- A physical iPhone device (HealthKit is limited in simulators)
- A signed-in iCloud account
- Health app configured and available
- Supported region and device capabilities

Some data types, such as heart rate variability, require:
- Apple Watch hardware
- Specific sensor support
- System-determined measurement conditions

---

### 3.5 iOS and watchOS Target Configuration

For apps that include Apple Watch support:

- The iOS app:
  - Requests authorization
  - Queries and writes most HealthKit data
- The watchOS app:
  - Collects sensor data
  - May write samples during workouts or sessions

Both targets must:
- Enable the HealthKit capability
- Use compatible bundle identifiers
- Be signed with the same development team

---

### 3.6 Verifying Setup

Before implementing HealthKit logic, verify:
- HealthKit capability is enabled
- Entitlements are present in the built app
- Privacy strings are defined
- Health data is available on the device

Early verification reduces time spent debugging authorization and data availability issues later in development.

---


## 4. Authorization and Privacy

HealthKit enforces a strict, user-controlled authorization model. Apps can only access health data types that the user has explicitly approved, and authorization can be changed or revoked at any time.

Understanding how authorization works is essential for designing reliable HealthKit features and setting correct user expectations.

---

### 4.1 Read and Write Permissions

HealthKit distinguishes between **reading** and **writing** data.

- **Read permission**
  - Allows an app to query existing health data
  - Does not allow modification or creation of data

- **Write permission**
  - Allows an app to save new health data to HealthKit
  - Does not imply read access to that data

Apps must request each permission explicitly for each data type they need.

---

### 4.2 Authorization Flow

The standard authorization flow is:

1. The app determines which data types it needs
2. The app requests authorization using `HKHealthStore`
3. The system presents an authorization UI to the user
4. The user grants or denies permissions
5. The app receives the result asynchronously

Authorization requests:
- Can only be initiated on iPhone
- Are presented by the system
- Cannot be customized or bypassed by the app

```swift
let healthStore = HKHealthStore()

healthStore.requestAuthorization(
    toShare: writeTypes,
    read: readTypes
) { success, error in
    // Handle result
}
```


### 4.3 Interpreting Authorization Results

A successful authorization request indicates that:
- The request was processed by the system
- The user was presented with the permission UI

It does not guarantee:
- That data exists for the requested types
- That data will be immediately available
- That future access will remain granted

Apps should always be prepared to handle empty results when querying HealthKit.

### 4.4 Checking Authorization Status

Apps can query the current authorization status for a specific data type.

Authorization status reflects:
- Whether access is allowed
- Whether access has been denied
- Whether the user has not yet made a choice

This allows apps to:
- Adjust UI based on access
- Prompt users to review permissions
- Avoid unnecessary authorization requests

### 4.5 Revocation and User Control

Users can modify HealthKit permissions at any time through the Health app.

User actions include:
- Revoking read or write access
- Changing source priorities
- Removing app data

Apps are not notified automatically when permissions change and must handle access checks dynamically.

### 4.6 Privacy Considerations

Health data is considered highly sensitive.

Best practices include:
- Requesting only the data types the app truly needs
- Explaining data usage clearly in privacy descriptions
- Avoiding unnecessary or repeated authorization prompts
- Handling health data securely within the app

Apps that misuse HealthKit permissions risk loss of user trust and App Review rejection.

### 4.7 Authorization in Multi-Device Apps

For apps with Apple Watch components:
- Authorization is always managed on iPhone
- Watch apps rely on permissions granted to the iOS app
- HealthKit access on Watch reflects iPhone authorization state

This separation should be considered when designing Watch-first experiences.


---

## 5. HealthKit Data Model

HealthKit uses a structured and extensible data model to represent health information from many different sources. Understanding this model is essential before working with queries, workouts, or advanced metrics such as HRV.

At a high level, HealthKit data is built on a small number of core types that layer responsibility and meaning.

### 5.1 HKObject

`HKObject` is the root class for all HealthKit data.

All HealthKit objects share common properties, including:
- A unique identifier
- Creation and modification dates
- Metadata
- Source information

Apps rarely work with `HKObject` directly, but every HealthKit sample ultimately inherits from it.

---

### 5.2 HKSample

`HKSample` represents a piece of health data that occurs over time.

Key characteristics:
- Has a start date and an end date
- Represents a measurement, event, or activity
- Can be queried and sorted by time

Examples of samples include:
- A heart rate measurement
- A sleep segment
- A workout session

Most HealthKit queries operate on `HKSample` subclasses.

---

### 5.3 HKQuantitySample

`HKQuantitySample` is the most commonly used sample type.

It represents numerical values measured with a specific unit.

Examples:
- Heart rate (beats per minute)
- Step count (count)
- Heart rate variability (milliseconds)
- Active energy burned (kilocalories)

Each quantity sample consists of:
- A value
- A unit
- A time range
- Optional metadata

```swift
let quantity = HKQuantity(
    unit: HKUnit.secondUnit(with: .milli),
    doubleValue: 42.0
)

let sample = HKQuantitySample(
    type: hrvType,
    quantity: quantity,
    start: startDate,
    end: endDate
)
```

### 5.4 Units and Normalization

HealthKit uses explicit units to normalize data across sources.

Important points:
- Units are not implicit
- The same data type may be queried in different units
- HealthKit handles conversion internally

For example:
- Heart rate may be stored as count per minute
- HRV is typically stored in milliseconds
- Energy may be queried in kilocalories or joules

Apps should always specify units intentionally to avoid misinterpretation.

### 5.5 HKCategorySample

`HKCategorySample` represents data that falls into discrete categories rather than numeric values.

Examples include:
- Sleep analysis states
- Mindfulness sessions
- Menstrual flow classifications

Category samples use:
- A predefined integer value
- A time range
- Optional metadata

### 5.6 HKWorkout

`HKWorkout` represents a structured physical or mindful activity.

A workout:
- Has a defined activity type
- Has a start and end time
- May include associated samples such as heart rate or energy
- Acts as a container for related HealthKit data

Workouts are commonly used to enable extended background execution on Apple Watch.

### 5.7 Metadata and Source Attribution

Every HealthKit sample may include metadata and source information.

Metadata can describe:
- Measurement context
- Device details
- Algorithm versions
- User-entered versus sensor-collected data

Source attribution identifies:
- The app that created the data
- The device on which it was collected

This information is used by the system to prioritize and present data correctly.

### 5.8 Why the Data Model Matters

Designing with the HealthKit data model in mind ensures that:
- Data is saved in the correct form
- Queries return meaningful results
- Multiple data sources can coexist
- The app integrates cleanly with the Health app


## 6. Health Data Types

HealthKit defines health information through strongly typed data categories. Each data type determines how data is stored, queried, displayed, and prioritized by the system.

Choosing the correct data type is essential for correct behavior and App Store compliance.

### 6.1 Quantity Types

Quantity types represent **numeric measurements** associated with a unit.

Common characteristics:
- Stored as numbers with explicit units
- Can be aggregated, averaged, or summed
- Often collected automatically by sensors

Examples:
- Heart rate
- Heart rate variability (SDNN)
- Step count
- Active energy burned

Quantity types are represented by `HKQuantityType` and stored as `HKQuantitySample`.

---

### 6.2 Category Types

Category types represent **discrete states or classifications** rather than numeric values.

Common characteristics:
- Use predefined integer values
- Represent states over a time range
- Often system-defined

Examples:
- Sleep analysis
- Mindfulness sessions
- Menstrual flow
- Audio exposure events

Category types are represented by `HKCategoryType` and stored as `HKCategorySample`.

---

### 6.3 Workout Types

Workout types represent **structured activities** performed by the user.

Characteristics:
- Have a start and end time
- Are associated with an activity type
- Can contain related samples such as heart rate and energy

Examples:
- Running
- Walking
- Cycling
- Mind and body activities

Workouts are represented by `HKWorkout`.

---

### 6.4 Characteristic Types

Characteristic types describe **static user attributes**.

Characteristics:
- Do not change frequently
- Are read-only
- Provided by the system or user profile

Examples:
- Date of birth
- Biological sex
- Blood type
- Wheelchair use

Characteristic types are accessed via `HKCharacteristicType`.

---

### 6.5 Choosing the Correct Data Type

When deciding which data type to use:

- Use **quantity types** for measurable values
- Use **category types** for states or classifications
- Use **workouts** for time-bounded activities
- Use **characteristic types** for personal attributes

Using the wrong data type can result in:
- Incorrect Health app presentation
- Data being ignored or deprioritized
- App Review rejection

---

### 6.6 System Constraints

Not all data types can be:
- Written by third-party apps
- Collected manually
- Triggered programmatically

Availability depends on:
- Platform (iOS vs watchOS)
- Device capabilities
- System policies

Apps should always verify data type availability before requesting authorization.
