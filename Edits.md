
# SDN Smart Routing – Environment & Code Fix Report

## 1. Controller Used

This project uses the POX SDN controller:

https://github.com/MurphyMc/pox

Run the controller with:

```bash
python3 pox.py openflow.discovery forwarding.l2_learning SDN-Smart-Routing.Smart_Routing_Module
````

---

## 2. Python Environment Requirements

> ⚠️ This project is based on legacy SDN libraries and is not fully compatible with Python 3.12 without version constraints.

---

## Required Packages

Create a `requirements.txt`:

```txt
networkx>=2.0,<3.0
fnss==0.9.1
pyzmq
numpy>=1.21,<2.0
scipy>=1.7,<2.0
matplotlib
python-igraph
fastnumbers
mininet
```

---

## Installation Note (Tested Stable Combo)

```bash
pip install "fnss==0.9.1" "networkx>=2.0,<3.0"
```

---

## 3. Key Code Fixes

---

## 3.1 Edge Normalization (CRITICAL)

The graph is undirected, but dictionary keys depend on ordering.

### Problem

```python
(3, 2) != (2, 3)
```

### Solution

Always normalize edges:

```python
edge = tuple(sorted(Edges[i]))
```

---

## Helper Function (Recommended)

```python
def edge_key(e):
    return tuple(sorted(e))
```

Usage:

```python
edge = edge_key(Edges[i])
```

---

## 3.2 Locations Requiring Fix

Edge normalization must be applied in:

* CC computation loop
* MTBF computation loop
* MTTR computation loop
* Any access to:

  ```python
  Links_Lengths_Dictionary[Edges[i]]
  ```

---

## 3.3 Correct Implementations

### CC computation

```python
edge = tuple(sorted(Edges[i]))
cc.append(Links_Lengths_Dictionary[edge] / minimum)
```

---

### MTBF computation

```python
edge = tuple(sorted(Edges[i]))
MTBF.append((cc[i] * 365 * 24) / Links_Lengths_Dictionary[edge])
```

---

### MTTR computation

```python
edge = tuple(sorted(Edges[i]))
mttr = round(Links_Lengths_Dictionary[edge] * Gama[i])

if mttr < 1:
    mttr = 1

MTTR.append(mttr)
```

---

## 3.4 Link Initialization Fix

```python
edge = tuple(sorted(Edges[i]))

L.append(
    Links(
        Edges[i],
        i,
        Links_Lengths_Dictionary[edge],
        MTBF[i],
        MTTR[i],
        0, 0, 0, True
    )
)
```

---

## 3.5 Stochastic Variable Fixes (CRITICAL)

### Problem

numpy arrays used instead of scalars

---

### TTF Fix

```python
TTF = np.random.exponential(scale=L[i].MTBF)
L[i].Next_Failure = round(TTF) + 1
```

---

### Moderator Fix

```python
moderator = np.random.uniform(0.1, 0.9)
```

---

### Log-Normal Recovery Time

```python
Log_Normal = np.random.lognormal(mu, sig)

print(
    'The link', L[link].ID,
    'will wait up to', round(Log_Normal),
    'to get recovery'
)
```

---

### Second Failure Time (TTF2)

```python
TTF2 = np.random.exponential(scale=L[link_return].MTBF)
```

---

## 4. Root Cause Summary

Issues were caused by:

* Undirected graph edge ordering inconsistency
* Mixing numpy arrays and scalar values
* Deprecated SciPy API usage
* FNSS + NetworkX version mismatch
* Incompatibility with Python 3.12 ecosystem

---

## 5. Final Recommendation

For stable execution:

* Use Python ≤ 3.10 (recommended: 3.7)
* Normalize all edges using `tuple(sorted(edge))`
* Ensure all stochastic outputs are scalars
* Avoid numpy arrays in scheduling logic
* Use compatible versions of NetworkX and FNSS

---

