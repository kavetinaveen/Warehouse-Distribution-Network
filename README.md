# Single Warehouse Distribution Network — Vehicle Routing Optimization

Optimization-based daily vehicle routing and scheduling for warehouse-to-store deliveries, using Google OR-Tools and Haversine distance.

---

## Problem Description

A single warehouse serves multiple retail stores across cities. The system assigns vehicles to deliver store demand while respecting:

- **Vehicle capacity** (weight in kg)
- **Delivery time windows** at each store
- **Vehicle types**: Fixed (pre-contracted) vs Adhoc (on-demand, higher cost)

**Objective**: Minimize total cost, with strong preference for Fixed vehicles over Adhoc.

---

## Mathematical Model

### Sets

| Symbol | Description |
|--------|-------------|
| $N$ | Set of nodes: depot (0) and stores $\{1, 2, \ldots, n\}$ |
| $V$ | Set of vehicles $\{1, 2, \ldots, K\}$ |
| $V^F$ | Subset of Fixed vehicles |
| $V^A$ | Subset of Adhoc vehicles ($V = V^F \cup V^A$) |

### Parameters

| Symbol | Description |
|--------|-------------|
| $t_{ij}$ | Travel time (minutes) from node $i$ to node $j$, derived from Haversine distance and average speed |
| $[a_i, b_i]$ | Time window for node $i$: earliest and latest arrival (minutes from midnight) |
| $q_i$ | Demand at node $i$ (kg); $q_0 = 0$ at depot |
| $Q_k$ | Capacity of vehicle $k$ (kg) |
| $c_k$ | Fixed cost of using vehicle $k$; $c_k = 0$ for Fixed, $c_k = M$ (large penalty) for Adhoc |
| $T_{\max}$ | Maximum route duration (minutes), e.g. 12 hours |

### Decision Variables

| Symbol | Type | Description |
|--------|------|-------------|
| $x_{ij}^k$ | Binary | 1 if vehicle $k$ travels from $i$ to $j$, 0 otherwise |
| $y^k$ | Binary | 1 if vehicle $k$ is used, 0 otherwise |
| $A_i^k$ | Continuous | Arrival time of vehicle $k$ at node $i$ |
| $L_i^k$ | Continuous | Cumulative load of vehicle $k$ when leaving node $i$ |

### Objective Function

```math
\min \quad \sum_{k \in V} c_k \cdot y^k + \sum_{k \in V} \sum_{(i,j)} t_{ij} \cdot x_{ij}^k
```

- **First term**: Fixed cost for using vehicles (Adhoc vehicles have high $c_k$, so their use is penalized).
- **Second term**: Total travel time (arc cost) over all routes.

By setting $c_k \gg \sum_{(i,j)} t_{ij}$ for Adhoc vehicles, the model effectively **minimizes the number of Adhoc vehicles used**, then minimizes total travel time among solutions with the same Adhoc count.

### Constraints

**Flow conservation (each store visited at most once):**

```math
\sum_{k \in V} \sum_{j \in N} x_{ij}^k = 1 \quad \forall i \in N \setminus \{0\}
```

**Depot flow (vehicles start and end at depot):**

```math
\sum_{j \in N \setminus \{0\}} x_{0j}^k = y^k \quad \forall k \in V
```

```math
\sum_{i \in N \setminus \{0\}} x_{i0}^k = y^k \quad \forall k \in V
```

**Route continuity:**
```math
\sum_{j \in N} x_{ij}^k = \sum_{j \in N} x_{ji}^k \quad \forall i \in N, \, k \in V
```

**Time windows:**
```math
a_i \leq A_i^k \leq b_i \quad \forall i \in N, \, k \in V \text{ (if } x_{ij}^k = 1 \text{ for some } j \text{)}
```

**Capacity:**
```math
L_i^k + q_j \leq Q_k \quad \text{(when } k \text{ delivers to } j \text{ after } i \text{)}
```

```math
L_0^k = 0, \quad L_i^k \geq 0 \quad \forall i \in N, \, k \in V
```

**Route duration:**
```math
A_{\mathrm{end}}^k - A_{\mathrm{start}}^k \leq T_{\max} \quad \forall k \in V
```

---

## Distance Calculation: Haversine Formula

Travel times are derived from geographic distance using the Haversine formula:

```math
d_{ij} = 2R \cdot \operatorname{arctan2}\left( \sqrt{a}, \sqrt{1-a} \right)
```

where

```math
a = \sin^2\left(\frac{\Delta\phi}{2}\right) + \cos(\phi_i) \cos(\phi_j) \sin^2\left(\frac{\Delta\lambda}{2}\right)
```

- $R = 6371$ km (Earth radius)
- $\phi_i, \lambda_i$ = latitude and longitude of node $i$ (radians)
- $\Delta\phi = \phi_j - \phi_i$, $\Delta\lambda = \lambda_j - \lambda_i$

Travel time (minutes):

```math
t_{ij} = \frac{d_{ij}}{v} \times 60 + s_j
```

- $v$ = average speed (km/h), e.g. 30
- $s_j$ = service time at node $j$ (minutes), e.g. 15 at stores, 0 at depot

---

## Data Requirements

| Source | Contents |
|--------|----------|
| **Store Lat Long** | Store name, coordinates (lat, lon) |
| **WH Lat Long** | Warehouse name, coordinates |
| **Vehicles** | Type (Fixed/Adhoc), count, capacity (kg), location |
| **Fixed Slot** | Store, city, time window (start, end) |
| **Last 15 Days Demand** | Date, store, weight (kg), delivery time window, city |

---

## Project Structure

```
VRP/
├── README.md
├── data/
│   └── Logistic Details_JH_Last 15 Days.xlsx
└── src/
    └── notebooks/
        └── explore_data.ipynb   # VRP implementation
```

---

## How to Run

1. Install dependencies: `pandas`, `openpyxl`, `ortools`, `matplotlib`
2. Open `src/notebooks/explore_data.ipynb`
3. Run all cells; the notebook loads data, builds the model, and solves with OR-Tools
4. Set `TARGET_DATE` and `MAX_STORES` (optional) for the day and store subset to optimize
5. Set `ACTUAL_ADHOC_USED` in the comparison cell if you have historical adhoc counts

---

## References

- [OR-Tools Vehicle Routing](https://developers.google.com/optimization/routing)
- [VRP with Time Windows](https://developers.google.com/optimization/routing/vrptw)
- [Capacitated VRP](https://developers.google.com/optimization/routing/cvrp)
