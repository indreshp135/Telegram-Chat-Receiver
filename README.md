```py

import numpy as np
from collections import Counter
from random import sample, seed
from math import ceil

def z_score(data, threshold=3):
    """
    Detects outliers using the Z-score method.
    Returns row indices of outliers.
    """
    mean = np.mean(data, axis=0)
    std_dev = np.std(data, axis=0)
    z_scores = np.abs((data - mean) / std_dev)
    return np.where(np.any(z_scores > threshold, axis=1))[0]

def iqr(data, threshold=1.5):
    """
    Detects outliers using the Interquartile Range (IQR) method.
    Returns row indices of outliers.
    """
    q1 = np.percentile(data, 25, axis=0)
    q3 = np.percentile(data, 75, axis=0)
    iqr = q3 - q1
    lower_bound = q1 - (threshold * iqr)
    upper_bound = q3 + (threshold * iqr)
    return np.where((data < lower_bound) | (data > upper_bound))[0]

def get_path_length(x, tree):
    path = []
    while tree:
        path.append(tree)
        if x < tree[0]:
            tree = tree[1]
        else:
            tree = tree[2]
    return len(path)

def build_tree(data):
    if not data:
        return None
    seed(0)
    index = sample(range(len(data)), 1)[0]
    value = data[index]
    left = [x for x in data if x < value]
    right = [x for x in data if x > value]
    return (value, build_tree(left), build_tree(right))

def isolation_forest(data, n_trees=100, sample_size=256):
    """
    Detects outliers using the Isolation Forest algorithm.
    Returns row indices of outliers.
    """
    trees = [build_tree(sample(data, sample_size)) for _ in range(n_trees)]
    path_lengths = [get_path_length(x, tree) for tree in trees for x in data]
    avg_path_lengths = np.mean(path_lengths)
    scores = [avg_path_lengths / path_length for path_length in path_lengths]
    threshold = np.percentile(scores, 100 * (1 - (1 / sample_size)))
    return np.where(np.array(scores) < threshold)[0]

def local_outlier_factor(data, k=5):
    """
    Detects outliers using the Local Outlier Factor (LOF) algorithm.
    Returns row indices of outliers.
    """
    n = len(data)
    lrd = np.zeros(n)
    lof = np.zeros(n)

    for i in range(n):
        distances = [np.linalg.norm(data[i] - data[j]) for j in range(n) if i != j]
        k_distances = sorted(distances)[:k]
        lrd[i] = np.mean(k_distances)

    for i in range(n):
        neighbors = [j for j in range(n) if np.linalg.norm(data[i] - data[j]) <= lrd[i]]
        lrdp = sum(lrd[j] for j in neighbors) / len(neighbors)
        lof[i] = lrdp / lrd[i] if lrd[i] > 0 else 1

    threshold = np.median(lof) * 3
    return np.where(lof > threshold)[0]

def knn_outlier_detection(data, k=5, contamination=0.1):
    """
    Detects outliers using the K-Nearest Neighbors (KNN) algorithm.
    Returns row indices of outliers.
    """
    n = len(data)
    distances = np.zeros((n, n))

    for i in range(n):
        for j in range(n):
            distances[i, j] = np.linalg.norm(data[i] - data[j])

    neighbors = [sorted(distances[i])[1:k+1] for i in range(n)]
    avg_neighbor_distances = [sum(neighbor) / k for neighbor in neighbors]
    threshold = np.percentile(avg_neighbor_distances, 100 * (1 - contamination))
    return np.where(np.array(avg_neighbor_distances) > threshold)[0]

def density_based_outlier_detection(data, eps=0.5, min_samples=5):
    """
    Detects outliers using the Density-Based Outlier Detection algorithm.
    Returns row indices of outliers.
    """
    n = len(data)
    neighbors = [[] for _ in range(n)]
    outliers = [True] * n

    for i in range(n):
        for j in range(n):
            if i != j and np.linalg.norm(data[i] - data[j]) <= eps:
                neighbors[i].append(j)

    for i in range(n):
        if len(neighbors[i]) >= min_samples:
            outliers[i] = False
            for j in neighbors[i]:
                outliers[j] = False

    return np.where(outliers)[0]

```
