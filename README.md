```py

import pandas as pd
from random import sample, seed
from math import ceil

def z_score(df, column, threshold=3):
    """
    Detects outliers using the Z-score method.
    Returns row indices of outliers.
    """
    mean = df[column].mean()
    std_dev = df[column].std()
    z_scores = abs((df[column] - mean) / std_dev)
    return df[z_scores > threshold].index

def iqr(df, column, threshold=1.5):
    """
    Detects outliers using the Interquartile Range (IQR) method.
    Returns row indices of outliers.
    """
    q1 = df[column].quantile(0.25)
    q3 = df[column].quantile(0.75)
    iqr = q3 - q1
    lower_bound = q1 - (threshold * iqr)
    upper_bound = q3 + (threshold * iqr)
    return df[(df[column] < lower_bound) | (df[column] > upper_bound)].index

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

def isolation_forest(df, columns, n_trees=100, sample_size=256):
    """
    Detects outliers using the Isolation Forest algorithm.
    Returns row indices of outliers.
    """
    data = df[columns].values
    trees = [build_tree(sample(data, sample_size)) for _ in range(n_trees)]
    path_lengths = [get_path_length(x, tree) for tree in trees for x in data]
    avg_path_lengths = sum(path_lengths) / len(path_lengths)
    scores = [avg_path_lengths / path_length for path_length in path_lengths]
    threshold = pd.Series(scores).quantile(1 - (1 / sample_size))
    return df.loc[pd.Series(scores) < threshold, :].index

def local_outlier_factor(df, columns, k=5):
    """
    Detects outliers using the Local Outlier Factor (LOF) algorithm.
    Returns row indices of outliers.
    """
    data = df[columns].values
    n = len(data)
    lrd = pd.Series([0] * n)
    lof = pd.Series([0] * n)

    for i in range(n):
        distances = [(data[i] - data[j])**2 for j in range(n) if i != j]
        k_distances = sorted(distances)[:k]
        lrd[i] = sum(k_distances) / k

    for i in range(n):
        neighbors = [j for j in range(n) if sum((data[i] - data[j])**2) <= lrd[i]]
        lrdp = sum(lrd[neighbors]) / len(neighbors)
        lof[i] = lrdp / lrd[i] if lrd[i] > 0 else 1

    threshold = lof.median() * 3
    return df.loc[lof > threshold, :].index

def knn_outlier_detection(df, columns, k=5, contamination=0.1):
    """
    Detects outliers using the K-Nearest Neighbors (KNN) algorithm.
    Returns row indices of outliers.
    """
    data = df[columns].values
    n = len(data)
    distances = pd.DataFrame(0, index=range(n), columns=range(n))

    for i in range(n):
        for j in range(n):
            distances.iloc[i, j] = sum((data[i] - data[j])**2)

    neighbors = [sorted(distances.iloc[i])[1:k+1] for i in range(n)]
    avg_neighbor_distances = [sum(neighbor) / k for neighbor in neighbors]
    threshold = pd.Series(avg_neighbor_distances).quantile(1 - contamination)
    return df.loc[pd.Series(avg_neighbor_distances) > threshold, :].index

def density_based_outlier_detection(df, columns, eps=0.5, min_samples=5):
    """
    Detects outliers using the Density-Based Outlier Detection algorithm.
    Returns row indices of outliers.
    """
    data = df[columns].values
    n = len(data)
    neighbors = [[] for _ in range(n)]
    outliers = [True] * n

    for i in range(n):
        for j in range(n):
            if i != j and sum((data[i] - data[j])**2) <= eps:
                neighbors[i].append(j)

    for i in range(n):
        if len(neighbors[i]) >= min_samples:
            outliers[i] = False
            for j in neighbors[i]:
                outliers[j] = False

    return df.loc[outliers, :].index
```
