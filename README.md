# Dune

**Dune** is a small Rust web app offering Price Forward Curve (PFC) machinery, specifically for power market data. Users provide market data and Dune constructs an arbitrage-free forward price curve.

The web app offers example request data for each endpoint. Some of the examples are valid as-is, while others may require additional data points or adjustments to the input.

Access without a registered API key is very limited. API keys can be requested via E-Mail.

[Dune Web App](https://dune.sbcb.at/)

## API Overview

Dune's API is organized into two groups:

* **Core** endpoints handle the construction, transformation, projection, and composition of forward curves.
* **Shape** endpoints provide tools for modifying the shape of an existing curve using historical spot data or smoothing.

All API endpoints return a common response structure containing metadata, diagnostics, the resulting `main` curve data, and, where applicable, debug information.

### Core endpoints

| Endpoint                          | Purpose                                                                                                                                                                                                                                                                                                                                    |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `POST /v1/core/from-settlements`  | Build a forward curve directly from market settlement contracts. Dune completes the contract set where necessary, constructs the delivery grid, solves the PFC, and returns the resulting curve. Optional debug output exposes the intermediate contract-reduction and curve-construction process.                                         |
| `POST /v1/core/from-constraints`  | Build a forward curve from an already completed set of constraints contained in `main.completed_constraints`. This is useful when the constraint set has already been prepared or when a curve needs to be reconstructed from existing constraint data.                                                                                    |
| `POST /v1/core/project`           | Project an existing full curve onto a supplied set of completed constraints. This can be used to adjust a curve so that it satisfies the specified constraint prices while retaining its overall shape as closely as possible. The response reports the maximum price correction and whether the result is within the requested tolerance. |
| `POST /v1/core/compose-base-peak` | Combine separate base and peak curves into a single composed curve. Base and peak can be supplied either as contracts or as existing `main` curve data. A configurable peak window determines which hours are treated as peak.                                                                                                             |

### Shape endpoints

| Endpoint                | Purpose                                                                                                                                                                                                                                        |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /v1/shape/shift`  | Derive a new curve shape from historical spot-market data. The endpoint produces a shape target; it deliberately does not perform the final projection onto constraints. The resulting curve can subsequently be passed to `/v1/core/project`. |
| `POST /v1/shape/smooth` | Smooth an existing curve while taking the completed constraint prices into account. This can be used to remove unwanted short-term irregularities while maintaining the relevant constraint structure.                                         |

### Health endpoint

| Endpoint      | Purpose                                                             |
| ------------- | ------------------------------------------------------------------- |
| `GET /health` | Simple health check. Returns HTTP `200 OK` when the API is running. |

## Typical workflow

A typical workflow starts with market settlement data and progressively turns it into a usable forward curve.

### 1. Start with market settlements

The most direct entry point is:

```text
POST /v1/core/from-settlements
```

Provide the available settlement contracts, including their delivery periods and prices.

Dune uses these contracts to construct the underlying delivery grid and solve for the primitive prices of the forward curve. Where the supplied contracts do not directly provide a complete system, Dune can construct synthetic constraints as part of the process.

If the optional debug flag is enabled, the response also provides information about:

* the original contracts
* the resulting time grid
* redundant contracts that were removed
* the reduced contract set
* synthetic contracts that were added
* an overview of the resulting constraint system

This makes `from-settlements` useful both for producing a curve and for understanding how Dune arrived at it.

### 2. Inspect or modify the curve shape

Once a curve exists, the shape endpoints can be used for further processing. 

For example, historical spot data can be used with:

```text
POST /v1/shape/shift
```

This creates a shape target based on historical patterns, holidays, and the supplied configuration.

Alternatively:

```text
POST /v1/shape/smooth
```

can be used when the objective is to smooth an existing curve.

The shape operations are deliberately separated from the final constraint projection. This allows the user to manipulate the curve shape before enforcing the market constraints. With that, curve shaping can also happen outside of Dune.

### 3. Project the resulting shape onto the market constraints

After modifying the curve shape, use:

```text
POST /v1/core/project
```

to project the curve back onto the completed constraints.

This is particularly useful when a desired curve shape has been generated independently of the market settlements. The projection makes the curve consistent with the supplied constraint prices.

The endpoint also reports the maximum absolute price correction and whether the supplied curve satisfied the configured tolerance before the correction.

### 4. Work with already-completed constraints

If the constraint set is already known, the intermediate settlement-processing step can be skipped.

Use:

```text
POST /v1/core/from-constraints
```

to construct a curve directly from `main.completed_constraints`.

This is useful when constraints have been generated or stored previously and need to be turned back into a full forward curve.

### 5. Construct a base/peak curve

For markets where separate base and peak products are available, use:

```text
POST /v1/core/compose-base-peak
```

The endpoint accepts separate base and peak inputs and combines them into a single curve.

Each side can be supplied either as:

* a list of contracts, allowing Dune to solve that side from the underlying market data; or
* an existing `main` curve.

A peak window defines the hours that belong to the peak period.

When contract data is supplied and debug is enabled, the debug information is provided separately for the base and peak sides.

## How Dune can help

Dune is intended to separate several parts of the forward-curve workflow that are often mixed together:

1. **Market constraints** — settlement contracts provide the prices that the curve needs to respect.
2. **Curve construction** — Dune solves the underlying forward prices from those constraints.
3. **Curve shape** — historical data or smoothing can be used to influence the shape between the market constraints.
4. **Projection** — the resulting shape can be projected back onto the market constraints.
5. **Base/peak composition** — separate base and peak structures can be combined into a single curve.

This makes it possible to use Dune in different ways depending on the available data.

For example, a simple workflow might be:

```text
Settlement contracts
        │
        ▼
/v1/core/from-settlements
        │
        ▼
   Forward curve
        │
        ├───────────────┐
        ▼               ▼
/v1/shape/smooth   /v1/shape/shift
        │               │
        └───────┬───────┘
                ▼
        /v1/core/project
                │
                ▼
       Final forward curve
```

Another workflow, when the market constraints are already available, can be much shorter:

```text
Completed constraints
        │
        ▼
/v1/core/from-constraints
        │
        ▼
   Forward curve
```

And for a base/peak market:

```text
Base contracts ──► base curve ──┐
                                ├──► /v1/core/compose-base-peak
Peak contracts ──► peak curve ──┘
                                │
                                ▼
                         Composed curve
```

The web application provides example requests for these workflows, making it possible to experiment with the API interactively before integrating Dune into another application or data pipeline.
