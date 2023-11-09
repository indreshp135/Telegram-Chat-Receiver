
> **Indresh** - _(09/11/2023 18:27:13)_
```
23
```
---
    
> **Indresh** - _(09/11/2023 18:24:47)_
```
Ok
```
---
    
> **Indresh** - _(09/11/2023 18:23:56)_
```
Ii
```
---
    
> **Indresh** - _(09/11/2023 18:22:57)_
```
Kt
```
---
    
> **Indresh** - _(09/11/2023 18:19:47)_
```
We
```
---
    
> **Indresh** - _(09/11/2023 18:19:37)_
```
Iu
```
---
    
> **Indresh** - _(09/11/2023 18:17:50)_
```
Ho
```
---
    
> **Indresh** - _(08/11/2023 06:25:45)_
```
using System;
using System.Collections.Generic;
using System.Linq;

public class ObservationCounter
{
    private Dictionary<string, Dictionary<string, int>> _counts = new Dictionary<string, Dictionary<string, int>>();
    private Dictionary<Tuple<string, string>, int> _jointCounts = new Dictionary<Tuple<string, string>, int>();
    private Dictionary<Tuple<string, string>, int> _index = new Dictionary<Tuple<string, string>, int>();
    private Dictionary<string, int> _nObs = new Dictionary<string, int>();

    public Dictionary<string, Dictionary<string, int>> Counts
    {
        get { return new Dictionary<string, Dictionary<string, int>>(_counts); }
    }

    public Dictionary<Tuple<string, string>, int> JointCounts
    {
        get { return new Dictionary<Tuple<string, string>, int>(_jointCounts); }
    }

    public Dictionary<Tuple<string, string>, int> Index
    {
        get { return new Dictionary<Tuple<string, string>, int>(_index); }
    }

    public void Update(List<Dictionary<string, string>> observationList)
    {
        foreach (var observation in observationList)
        {
            Dictionary<string, string> obs = observation.Where(item => !IsNaN(item.Value)).ToDictionary(item => item.Key, item => item.Value);
            var obs1 = obs.ToList();
            var obs2 = obs.ToList();
            UpdateCounts(obs1);
            UpdateJointCounts(obs2);
        }
    }

    public int GetCount(Tuple<string, string> item)
    {
        string featureName = item.Item1;
        if (_counts.TryGetValue(featureName, out Dictionary<string, int> featureCounts))
        {
            if (featureCounts.TryGetValue(item.Item2, out int count))
            {
                return count;
            }
        }
        return 0;
    }

    private void UpdateCounts(List<KeyValuePair<string, string>> observation)
    {
        foreach (var item in observation)
        {
            string featureName = item.Key;
            if (!_counts.ContainsKey(featureName))
            {
                _counts[featureName] = new Dictionary<string, int>();
            }
            if (_index.TryAdd(item, 0))
            {
                _index[item] = 0;
            }
            if (!_nObs.ContainsKey(featureName))
            {
                _nObs[featureName] = 0;
            }
            _counts[featureName][item.Value] = _counts[featureName].GetValueOrDefault(item.Value) + 1;
            _nObs[featureName] = _nObs.GetValueOrDefault(featureName) + 1;
        }
    }

    private void UpdateJointCounts(List<KeyValuePair<string, string>> observations)
    {
        var pairs = observations.OrderBy(item => item.Key).Combinations(2).ToList();
        foreach (var pair in pairs)
        {
            var featureTuple1 = pair[0];
            var featureTuple2 = pair[1];
            var key = Tuple.Create(featureTuple1, featureTuple2);
            if (!_jointCounts.ContainsKey(key))
            {
                _jointCounts[key] = 0;
            }
            _jointCounts[key] += 1;
        }
    }

    public bool IsNaN(object x)
    {
        return !x.Equals(x);
    }
}

public static class Extensions
{
    public static List<List<T>> Combinations<T>(this IEnumerable<T> elements, int k)
    {
        List<List<T>> result = new List<List<T>>();
        if (k == 0 || elements.Count() == 0)
        {
            result.Add(new List<T>());
            return result;
        }
        var head = elements.First();
        var tail = elements.Skip(1);
        var withoutHead = tail.Combinations(k);
        foreach (var comb in withoutHead)
        {
            result.Add(comb);
        }
        var withHead = tail.Combinations(k - 1);
        foreach (var comb in withHead)
        {
            comb.Insert(0, head);
            result.Add(comb);
        }
        return result;
    }
}
```
---
    
> **Indresh** - _(08/11/2023 06:29:15)_
```
using System;
using System.Collections.Generic;
using System.Linq;

public class IncrementingDict<TKey>
{
    private Dictionary<TKey, int> dictionary = new Dictionary<TKey, int>();
    private int nextValue = 0;

    public void Insert(TKey key)
    {
        if (!dictionary.ContainsKey(key))
        {
            dictionary[key] = nextValue;
            nextValue++;
        }
    }

    public int this[TKey key]
    {
        get { return dictionary[key]; }
    }

    public int Count => dictionary.Count;

    public bool ContainsKey(TKey key)
    {
        return dictionary.ContainsKey(key);
    }
}

public class ObservationCounter
{
    private Dictionary<string, Dictionary<Tuple<string, string>, int>> counts = new Dictionary<string, Dictionary<Tuple<string, string>, int>();
    private Dictionary<Tuple<string, string>, int> jointCounts = new Dictionary<Tuple<string, string>, int>();
    private IncrementingDict<Tuple<string, string>> index = new IncrementingDict<Tuple<string, string>();
    private Dictionary<string, int> nObs = new Dictionary<string, int>();

    public Dictionary<string, Dictionary<Tuple<string, string>, int>> Counts => counts.ToDictionary(entry => entry.Key, entry => new Dictionary<Tuple<string, string>, int>(entry.Value));
    public Dictionary<Tuple<string, string>, int> JointCounts => new Dictionary<Tuple<string, string>, int>(jointCounts);
    public IncrementingDict<Tuple<string, string>> Index => index;
    
    public void Update(List<Dictionary<string, string>> observationList)
    {
        foreach (var observation in observationList)
        {
            var obs = observation.Where(entry => !IsNaN(entry.Value)).ToDictionary(entry => entry.Key, entry => entry.Value);
            UpdateCounts(obs);
            UpdateJointCounts(obs);
        }
    }

    public int GetCount(Tuple<string, string> item)
    {
        var featureName = GetFeatureName(item);
        if (counts.ContainsKey(featureName))
        {
            if (counts[featureName].ContainsKey(item))
            {
                return counts[featureName][item];
            }
        }
        return 0;
    }

    private void UpdateCounts(Dictionary<string, string> observation)
    {
        foreach (var item in observation)
        {
            var featureName = GetFeatureName(item);
            if (!counts.ContainsKey(featureName))
            {
                counts[featureName] = new Dictionary<Tuple<string, string>, int>();
            }
            if (!nObs.ContainsKey(featureName))
            {
                nObs[featureName] = 0;
            }
            counts[featureName][Tuple.Create(item.Key, item.Value)] = counts[featureName].ContainsKey(Tuple.Create(item.Key, item.Value))
                ? counts[featureName][Tuple.Create(item.Key, item.Value)] + 1
                : 1;
            index.Insert(Tuple.Create(item.Key, item.Value));
            nObs[featureName]++;
        }
    }

    private void UpdateJointCounts(Dictionary<string, string> observation)
    {
        var observations = observation.Select(item => Tuple.Create(item.Key, item.Value)).ToList();
        for (int i = 0; i < observations.Count; i++)
        {
            for (int j = i + 1; j < observations.Count; j++)
            {
                var pair = Tuple.Create(observations[i], observations[j]);
                if (jointCounts.ContainsKey(pair))
                {
                    jointCounts[pair]++;
                }
                else
                {
                    jointCounts[pair] = 1;
                }
            }
        }
    }

    private string GetFeatureName(Tuple<string, string> featureTuple)
    {
        return featureTuple.Item1;
    }

    private bool IsNaN(object x)
    {
        return !x.Equals(x);
    }
}
```
---
    
> **Indresh** - _(08/11/2023 06:35:21)_
```
using System;
using System.Collections.Generic;
using System.Linq;

public class IncrementingDict
{
    private Dictionary<string, int> dictionary = new Dictionary<string, int>();
    private int nextValue = 0;

    public void Insert(Tuple<string, string> key)
    {
        var keyString = $"{key.Item1}:{key.Item2}";
        if (!dictionary.ContainsKey(keyString))
        {
            dictionary[keyString] = nextValue;
            nextValue++;
        }
    }

    public int this[Tuple<string, string> key]
    {
        get
        {
            var keyString = $"{key.Item1}:{key.Item2}";
            return dictionary[keyString];
        }
    }

    public int Count => dictionary.Count;

    public bool ContainsKey(Tuple<string, string> key)
    {
        var keyString = $"{key.Item1}:{key.Item2}";
        return dictionary.ContainsKey(keyString);
    }
}

public class ObservationCounter
{
    private Dictionary<string, Dictionary<string, int>> counts = new Dictionary<string, Dictionary<string, int>();
    private Dictionary<string, int> jointCounts = new Dictionary<string, int>();
    private IncrementingDict index = new IncrementingDict();
    private Dictionary<string, int> nObs = new Dictionary<string, int>();

    public Dictionary<string, Dictionary<string, int>> Counts => counts.ToDictionary(entry => entry.Key, entry => new Dictionary<string, int>(entry.Value));
    public Dictionary<string, int> JointCounts => new Dictionary<string, int>(jointCounts);
    public IncrementingDict Index => index;

    public void Update(List<Dictionary<string, string>> observationList)
    {
        foreach (var observation in observationList)
        {
            var obs = observation.Where(entry => !IsNaN(entry.Value)).ToDictionary(entry => entry.Key, entry => entry.Value);
            UpdateCounts(obs);
            UpdateJointCounts(obs);
        }
    }

    public int GetCount(Tuple<string, string> item)
    {
        var featureName = GetFeatureName(item);
        if (counts.ContainsKey(featureName))
        {
            var keyString = $"{item.Item1}:{item.Item2}";
            if (counts[featureName].ContainsKey(keyString))
            {
                return counts[featureName][keyString];
            }
        }
        return 0;
    }

    private void UpdateCounts(Dictionary<string, string> observation)
    {
        foreach (var item in observation)
        {
            var featureName = GetFeatureName(item);
            if (!counts.ContainsKey(featureName))
            {
                counts[featureName] = new Dictionary<string, int>();
            }
            if (!nObs.ContainsKey(featureName))
            {
                nObs[featureName] = 0;
            }
            var keyString = $"{item.Key}:{item.Value}";
            counts[featureName][keyString] = counts[featureName].ContainsKey(keyString)
                ? counts[featureName][keyString] + 1
                : 1;
            index.Insert(Tuple.Create(item.Key, item.Value));
            nObs[featureName]++;
        }
    }

    private void UpdateJointCounts(Dictionary<string, string> observation)
    {
        var observations = observation.Select(item => $"{item.Key}:{item.Value}").ToList();
        for (int i = 0; i < observations.Count; i++)
        {
            for (int j = i + 1; j < observations.Count; j++)
            {
                var pair = $"{observations[i]}:{observations[j]}";
                if (jointCounts.ContainsKey(pair))
                {
                    jointCounts[pair]++;
                }
                else
                {
                    jointCounts[pair] = 1;
                }
            }
        }
    }

    private string GetFeatureName(Tuple<string, string> featureTuple)
    {
        return featureTuple.Item1;
    }

    private bool IsNaN(object x)
    {
        return !x.Equals(x);
    }
}
```
---
    
> **Indresh** - _(08/11/2023 06:45:04)_
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Collections;
using System.Tuple;

public class IncrementingDict : IDictionary
{
    private Dictionary<object, int> _d = new Dictionary<object, int>();
    private int _nextVal = 0;

    public void Add(object key, object value)
    {
        if (_d.ContainsKey(key))
        {
            return;
        }
        _d[key] = _nextVal;
        _nextVal++;
    }

    // Implement other IDictionary methods as needed

    public int Count
    {
        get { return _d.Count; }
    }

    // Implement other IDictionary properties as needed
}

public class ObservationCounter
{
    private Dictionary<string, int> _nObs = new Dictionary<string, int>();
    private Dictionary<string, Dictionary<Tuple<string, string>, int>> _counts = new Dictionary<string, Dictionary<Tuple<string, string>, int>>();
    private Dictionary<Tuple<string, string>, int> _jointCounts = new Dictionary<Tuple<string, string>, int>();
    private IncrementingDict _index = new IncrementingDict();

    public Dictionary<string, Dictionary<Tuple<string, string>, int>> Counts
    {
        get { return _counts.ToDictionary(entry => entry.Key, entry => entry.Value.ToDictionary(x => Tuple.Create(x.Key, x.Value), x => x.Value)); }
    }

    public Dictionary<Tuple<string, string>, int> JointCounts
    {
        get { return _jointCounts.ToDictionary(entry => Tuple.Create(entry.Key.Item1, entry.Key.Item2), entry => entry.Value); }
    }

    public IncrementingDict Index
    {
        get { return _index; }
    }

    public void Update(IEnumerable<Dictionary<string, string>> observationEnumerable)
    {
        foreach (var observation in observationEnumerable)
        {
            var obs = observation.Where(x => !IsNaN(x.Value)).ToDictionary(entry => entry.Key, entry => entry.Value);
            UpdateCounts(obs);
            UpdateJointCounts(obs);
        }
    }

    public int GetCount(Tuple<string, string> item)
    {
        string featureName = item.Item1;
        if (_counts.ContainsKey(featureName))
        {
            if (_counts[featureName].ContainsKey(item))
            {
                return _counts[featureName][item];
            }
        }
        return 0;
    }

    private void UpdateCounts(Dictionary<string, string> observation)
    {
        foreach (var item in observation)
        {
            string featureName = item.Key;
            if (!_counts.ContainsKey(featureName))
            {
                _counts[featureName] = new Dictionary<Tuple<string, string>>();
            }
            if (_counts[featureName].ContainsKey(Tuple.Create(item.Key, item.Value)))
            {
                _counts[featureName][Tuple.Create(item.Key, item.Value)]++;
            }
            else
            {
                _counts[featureName][Tuple.Create(item.Key, item.Value)] = 1;
            }
            _index.Add(Tuple.Create(item.Key, item.Value));
            if (_nObs.ContainsKey(featureName))
            {
                _nObs[featureName]++;
            }
            else
            {
                _nObs[featureName] = 1;
            }
        }
    }

    private void UpdateJointCounts(Dictionary<string, string> observation)
    {
        var pairs = observation.Keys.ToList().OrderBy(x => x).ToList();
        for (int i = 0; i < pairs.Count - 1; i++)
        {
            for (int j = i + 1; j < pairs.Count; j++)
            {
                Tuple<string, string> pair = Tuple.Create(pairs[i], pairs[j]);
                if (_jointCounts.ContainsKey(pair))
                {
                    _jointCounts[pair]++;
                }
                else
                {
                    _jointCounts[pair] = 1;
                }
            }
        }
    }

    private bool IsNaN(object x)
    {
        if (x is double)
        {
            return double.IsNaN((double)x);
        }
        return x != x;
    }
}
```
---
    
> **Indresh** - _(08/11/2023 07:14:53)_
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Numerics;
using MathNet.Numerics.LinearAlgebra;
using MathNet.Numerics.LinearAlgebra.Double;

public static class RandomWalkUtility
{
    private const double EPS = 1e-8; // tolerance below which to consider a value as zero

    public static DenseVector RandomWalk(
        SparseMatrix transitionMatrix,
        double alpha,
        double errTol,
        int maxIter)
    {
        int n = transitionMatrix.RowCount;
        DenseVector dampingVec = (1.0 - alpha) / n * DenseVector.Create(n, _ => 1.0);
        DenseVector pi = 1.0 / n * DenseVector.Create(n, _ => 1.0);

        for (int iter = 0; iter < maxIter; iter++)
        {
            DenseVector piNext = dampingVec + alpha * transitionMatrix.TransposeThisAndMultiply(pi);
            double error = pi.Subtract(piNext).LInfinityNorm();
            pi = piNext;
            if (error <= errTol)
            {
                break;
            }
        }

        double piSum = pi.Sum();
        if (piSum < EPS)
        {
            throw new ArgumentException("Stationary probabilities sum approximately zero");
        }

        return pi.Divide(piSum);
    }

    public static SparseMatrix<double> DictToCsrMatrix(
        Dictionary<Tuple<int, int>, double> dataDict,
        Tuple<int, int> shape)
    {
        if (dataDict.Count == 0)
        {
            throw new ArgumentException("Dictionary must not be empty");
        }

        int numRows = shape.Item1;
        int numCols = shape.Item2;
        if (numRows <= 0 || numCols <= 0)
        {
            throw new ArgumentException("Shape dimensions must be positive integers");
        }

        var values = dataDict.Values.ToList();
        var rowIndices = dataDict.Keys.Select(k => k.Item1).ToArray();
        var colIndices = dataDict.Keys.Select(k => k.Item2).ToArray();

        return SparseMatrix.OfIndexed(numRows, numCols, rowIndices, colIndices, values);
    }

    public static SparseMatrix<double> RowNormalizeCsrMatrix(SparseMatrix<double> matrix)
    {
        if (matrix == null)
        {
            throw new ArgumentNullException("Input matrix is null");
        }

        if (matrix.RowCount <= 0 || matrix.ColumnCount <= 0)
        {
            throw new ArgumentException("Input matrix dimensions must be positive");
        }

        if (matrix.NonZerosCount == 0)
        {
            throw new ArgumentException("Input matrix must not store zeros");
        }

        var rowSums = matrix.RowSums().ToColumnMatrix();
        var normalizedData = matrix.EnumerateIndexed(Zeros.AllowSkip, DenseMatrixColumnMajor.StorageFormat)
            .Select(entry => entry.Value / rowSums[entry.RowIndex, 0]);

        return SparseMatrix.OfIndexed(matrix.RowCount, matrix.ColumnCount, matrix.EnumerateIndexed(Zeros.AllowSkip), normalizedData);
    }
}
```
---
    
> **Indresh** - _(08/11/2023 09:26:01)_
```
using System;
using System.Collections.Generic;
using System.Linq;
using MathNet.Numerics.LinearAlgebra;
using MathNet.Numerics.LinearAlgebra.Double;
using MathNet.Numerics.LinearAlgebra.Factorization;
using MathNet.Numerics.LinearAlgebra.Storage;

public class RandomWalk
{
    public static Vector<double> RunRandomWalk(SparseMatrix transitionMatrix, double alpha, double errTol, int maxIter)
    {
        int n = transitionMatrix.RowCount;
        Vector<double> dampingVec = (1 - alpha) / n * Vector<double>.Build.Dense(n, 1.0);
        Vector<double> pi = 1.0 / n * Vector<double>.Build.Dense(n, 1.0);

        for (int iter = 0; iter < maxIter; iter++)
        {
            Vector<double> piNext = dampingVec + alpha * transitionMatrix.TransposeThisAndMultiply(pi);
            double err = pi.Subtract(piNext).LInfinityNorm();
            pi = piNext;
            if (err <= errTol)
            {
                break;
            }
        }

        double piSum = pi.Sum();
        if (piSum < 1e-8)
        {
            throw new Exception("Stationary probabilities sum approximately zero");
        }

        return pi / piSum;
    }

    public static SparseMatrix<double> DictToSparseMatrix(Dictionary<Tuple<int, int>, double> dataDict, Tuple<int, int> shape)
    {
        if (dataDict.Count == 0)
        {
            throw new Exception("Dictionary must not be empty");
        }

        double[] data = dataDict.Values.ToArray();
        var rowIndex = dataDict.Keys.Select(key => key.Item1).ToArray();
        var colIndex = dataDict.Keys.Select(key => key.Item2).ToArray();
        var storage = new CompressedSparseRowMatrixStorage<double>(data, rowIndex, colIndex, shape.Item1, shape.Item2);
        return new SparseMatrix(storage);
    }

    public static SparseMatrix<double> RowNormalizeSparseMatrix(SparseMatrix<double> matrix)
    {
        if (matrix.RowCount != matrix.ColumnCount)
        {
            throw new Exception("Input matrix must be square");
        }

        var rowSums = matrix.RowSums();
        for (int i = 0; i < rowSums.Count; i++)
        {
            if (rowSums[i] == 0.0)
            {
                throw new Exception("Input matrix must not store zeros");
            }
        }

        var normalizedData = matrix.EnumerateNonZero().Select((entry, index) => entry.Value / rowSums[entry.Row])
            .ToArray();

        var rowIndices = matrix.EnumerateNonZero().Select(entry => entry.Row).ToArray();
        var colIndices = matrix.EnumerateNonZero().Select(entry => entry.Column).ToArray();

        return SparseMatrix.OfIndexed(shape: matrix.RowCount, matrix.ColumnCount, rowIndices, colIndices, normalizedData);
    }
}
```
---
    
> **Indresh** - _(08/11/2023 09:56:16)_
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Numerics;

namespace CoupledBiasedRandomWalks
{
    class CBRW
    {
        // Random walk parameters
        private readonly Dictionary<string, float> PRESET_RW_PARAMS = new Dictionary<string, float>
        {
            { "alpha", 0.95f },
            { "err_tol", 1e-3f },
            { "max_iter", 100 }
        };

        private Dictionary<string, float> rwParams;
        private bool ignoreUnknown;
        private float unknownFeatureScore;

        private Dictionary<string, int> counter;

        private Dictionary<string, float> stationaryProb;
        private Dictionary<string, float> featureRelevance;

        public CBRW(Dictionary<string, float> rwParams = null, bool ignoreUnknown = false)
        {
            this.rwParams = rwParams ?? new Dictionary<string, float>(PRESET_RW_PARAMS);
            this.ignoreUnknown = ignoreUnknown;
            this.unknownFeatureScore = ignoreUnknown ? 0.0f : float.NaN;
            this.counter = new Dictionary<string, int>();
            this.stationaryProb = null;
            this.featureRelevance = null;
        }

        public Dictionary<string, float> FeatureWeights => this.featureRelevance;

        public CBRW AddObservations(List<Dictionary<string, object>> observationList)
        {
            foreach (var observation in observationList)
            {
                UpdateCounter(observation);
            }
            return this;
        }

        public CBRW Fit()
        {
            int nObserved = GetMode(this.counter.Values);
            if (nObserved == 0)
            {
                throw new CBRWFitException("Must add observations before calling fit method.");
            }

            try
            {
                var pi = RandomWalk(ComputeBiasedTransitionMatrix(), this.rwParams);
                this.stationaryProb = new Dictionary<string, float>();
                var featureRelevance = new Dictionary<string, float>();

                foreach (var kvp in this.counter)
                {
                    var feature = kvp.Key;
                    var idx = kvp.Value;
                    var prob = pi[idx];
                    this.stationaryProb[feature] = prob;
                    featureRelevance[GetFeatureName(feature)] += prob;
                }

                var featureRelSum = featureRelevance.Values.Sum();
                if (featureRelSum < float.Epsilon)
                {
                    throw new CBRWFitException("Feature weights sum is approximately zero.");
                }

                featureRelevance = featureRelevance.ToDictionary(kvp => kvp.Key, kvp => kvp.Value / featureRelSum);

                this.featureRelevance = featureRelevance;
            }
            catch (Exception err)
            {
                throw new CBRWFitException(err.Message);
            }

            return this;
        }

        public float[] Score(List<Dictionary<string, object>> observationList)
        {
            if (this.featureRelevance == null || this.stationaryProb == null)
            {
                throw new CBRWScoreException("Must call fit method to train on added observations before scoring.");
            }

            return observationList.Select(observation => Score(observation)).ToArray();
        }

        private float Score(Dictionary<string, object> observation)
        {
            return ValueScores(observation).Values.Sum();
        }

        public List<Dictionary<string, float>> ValueScores(List<Dictionary<string, object>> observationList)
        {
            if (this.featureRelevance == null || this.stationaryProb == null)
            {
                throw new CBRWScoreException("Must call fit method to train on added observations before scoring.");
            }

            return observationList.Select(observation => ValueScores(observation)).ToList();
        }
```
---
    
> **Indresh** - _(08/11/2023 09:56:16)_
```
private Dictionary<string, float> ValueScores(Dictionary<string, object> observation)
        {
            return observation.ToDictionary(
                kvp => GetFeatureName(kvp.Key),
                kvp => GetFeatureRelevance(kvp.Key) * this.stationaryProb.GetValueOrDefault(kvp.Key, this.unknownFeatureScore)
            );
        }

        private float GetFeatureRelevance(string feature)
        {
            var featureName = GetFeatureName(feature);
            return this.featureRelevance.GetValueOrDefault(featureName, 0);
        }

        private float[] RandomWalk(CSRMatrix transitionMatrix, Dictionary<string, float> rwParams)
        {
            // Implement your random walk logic here
            // This involves performing matrix operations and iteration based on the given parameters
            // You'll need to use external libraries for matrix operations
            // Return the result as a float array
            throw new NotImplementedException();
        }

        private CSRMatrix ComputeBiasedTransitionMatrix()
        {
            // Implement your logic to compute the biased transition matrix
            // This involves calculations based on the counter data
            // You'll need external libraries for matrix operations
            // Return the result as a CSRMatrix
            throw new NotImplementedException();
        }

        private void UpdateCounter(Dictionary<string, object> observation)
        {
            // Implement your logic to update the counter based on the observation
            // This involves updating the counter dictionary with counts
            throw new NotImplementedException();
        }

        private string GetFeatureName(string feature)
        {
            // Implement your logic to extract the feature name from the feature string
            throw new NotImplementedException();
        }

        private int GetMode(IEnumerable<int> values)
        {
            // Implement your logic to calculate the mode from a collection of values
            throw new NotImplementedException();
        }
    }

    class CBRWError : Exception
    {
        public CBRWError(string message) : base(message) { }
    }

    class CBRWFitException : CBRWError
    {
        public CBRWFitException(string message) : base(message) { }
    }

    class CBRWScoreException : CBRWError
    {
        public CBRWScoreException(string message) : base(message) { }
    }
}
```
---
    
> **Indresh** - _(08/11/2023 09:57:26)_
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Numerics;

namespace CoupledBiasedRandomWalks
{
    class CBRW
    {
        // Random walk parameters
        private readonly Dictionary<string, float> PRESET_RW_PARAMS = new Dictionary<string, float>
        {
            { "alpha", 0.95f },
            { "err_tol", 1e-3f },
            { "max_iter", 100 }
        };

        private Dictionary<string, float> rwParams;
        private bool ignoreUnknown;
        private float unknownFeatureScore;

        private Dictionary<string, int> counter;

        private Dictionary<string, float> stationaryProb;
        private Dictionary<string, float> featureRelevance;

        public CBRW(Dictionary<string, float> rwParams = null, bool ignoreUnknown = false)
        {
            this.rwParams = rwParams ?? new Dictionary<string, float>(PRESET_RW_PARAMS);
            this.ignoreUnknown = ignoreUnknown;
            this.unknownFeatureScore = ignoreUnknown ? 0.0f : float.NaN;
            this.counter = new Dictionary<string, int>();
            this.stationaryProb = null;
            this.featureRelevance = null;
        }

        public Dictionary<string, float> FeatureWeights => this.featureRelevance;

        public CBRW AddObservations(List<Dictionary<string, object>> observationList)
        {
            foreach (var observation in observationList)
            {
                UpdateCounter(observation);
            }
            return this;
        }

        public CBRW Fit()
        {
            int nObserved = GetMode(this.counter.Values);
            if (nObserved == 0)
            {
                throw new CBRWFitException("Must add observations before calling fit method.");
            }

            try
            {
                var pi = RandomWalk(ComputeBiasedTransitionMatrix(), this.rwParams);
                this.stationaryProb = new Dictionary<string, float>();
                var featureRelevance = new Dictionary<string, float>();

                foreach (var kvp in this.counter)
                {
                    var feature = kvp.Key;
                    var idx = kvp.Value;
                    var prob = pi[idx];
                    this.stationaryProb[feature] = prob;
                    featureRelevance[GetFeatureName(feature)] += prob;
                }

                var featureRelSum = featureRelevance.Values.Sum();
                if (featureRelSum < float.Epsilon)
                {
                    throw new CBRWFitException("Feature weights sum is approximately zero.");
                }

                featureRelevance = featureRelevance.ToDictionary(kvp => kvp.Key, kvp => kvp.Value / featureRelSum);

                this.featureRelevance = featureRelevance;
            }
            catch (Exception err)
            {
                throw new CBRWFitException(err.Message);
            }

            return this;
        }

        public float[] Score(List<Dictionary<string, object>> observationList)
        {
            if (this.featureRelevance == null || this.stationaryProb == null)
            {
                throw new CBRWScoreException("Must call fit method to train on added observations before scoring.");
            }

            return observationList.Select(observation => Score(observation)).ToArray();
        }

        private float Score(Dictionary<string, object> observation)
        {
            return ValueScores(observation).Values.Sum();
        }

        public List<Dictionary<string, float>> ValueScores(List<Dictionary<string, object>> observationList)
        {
            if (this.featureRelevance == null || this.stationaryProb == null)
            {
                throw new CBRWScoreException("Must call fit method to train on added observations before scoring.");
            }

            return observationList.Select(observation => ValueScores(observation)).ToList();
        }
```
---
    
> **Indresh** - _(08/11/2023 09:57:26)_
```
private Dictionary<string, float> ValueScores(Dictionary<string, object> observation)
        {
            return observation.ToDictionary(
                kvp => GetFeatureName(kvp.Key),
                kvp => GetFeatureRelevance(kvp.Key) * this.stationaryProb.GetValueOrDefault(kvp.Key, this.unknownFeatureScore)
            );
        }

        private float GetFeatureRelevance(string feature)
        {
            var featureName = GetFeatureName(feature);
            return this.featureRelevance.GetValueOrDefault(featureName, 0);
        }

        private float[] RandomWalk(CSRMatrix transitionMatrix, Dictionary<string, float> rwParams)
        {
            // Implement your random walk logic here
            // This involves performing matrix operations and iteration based on the given parameters
            // You'll need to use external libraries for matrix operations
            // Return the result as a float array
            throw new NotImplementedException();
        }

        private CSRMatrix ComputeBiasedTransitionMatrix()
        {
            // Implement your logic to compute the biased transition matrix
            // This involves calculations based on the counter data
            // You'll need external libraries for matrix operations
            // Return the result as a CSRMatrix
            throw new NotImplementedException();
        }

        private void UpdateCounter(Dictionary<string, object> observation)
        {
            // Implement your logic to update the counter based on the observation
            // This involves updating the counter dictionary with counts
            throw new NotImplementedException();
        }

        private string GetFeatureName(string feature)
        {
            // Implement your logic to extract the feature name from the feature string
            throw new NotImplementedException();
        }

        private int GetMode(IEnumerable<int> values)
        {
            // Implement your logic to calculate the mode from a collection of values
            throw new NotImplementedException();
        }
    }

    class CBRWError : Exception
    {
        public CBRWError(string message) : base(message) { }
    }

    class CBRWFitException : CBRWError
    {
        public CBRWFitException(string message) : base(message) { }
    }

    class CBRWScoreException : CBRWError
    {
        public CBRWScoreException(string message) : base(message) { }
    }
}
```
---
    
> **Indresh** - _(08/11/2023 10:10:02)_
```
using System.Collections.Generic;
using System.Linq;

public int GetMode(Dictionary<string, int> counter)
{
    if (counter.Count == 0)
    {
        return 0;
    }

    var maxCount = counter.Values.Max();
    var mode = counter.FirstOrDefault(entry => entry.Value == maxCount);
    
    if (mode.Value != null)
    {
        return mode.Value;
    }

    return 0;
}
```
---
    
> **Indresh** - _(09/11/2023 10:04:16)_
```
using System;
using System.Collections.Generic;

public class YourTestClass
{
    private Dictionary<string, Dictionary<string, List<Tuple<Tuple<string, string>, int>>>> table;
    private Dictionary<Tuple<string, string>, int> jointCounts;

    // Your existing methods and setup here...

    public void YourTestMethod()
    {
        Dictionary<string, Dictionary<string, List<Tuple<Tuple<string, string>, int>>>> table = new Dictionary<string, Dictionary<string, List<Tuple<Tuple<string, string>, int>>>>()
        {
            { "feature_a", new Dictionary<string, List<Tuple<Tuple<string, string>, int>>>()
                {
                    { "expected", new List<Tuple<Tuple<string, string>, int>>()
                        {
                            new Tuple<Tuple<string, string>, int>(Tuple.Create("feature_a", "a_val_1"), 2)
                        }
                    }
                }
            },
            { "feature_b", new Dictionary<string, List<Tuple<Tuple<string, string>, int>>>()
                {
                    { "expected", new List<Tuple<Tuple<string, string>, int>>()
                        {
                            new Tuple<Tuple<string, string>, int>(Tuple.Create("feature_b", "b_val_1"), 2)
                        }
                    }
                }
            },
            { "feature_c", new Dictionary<string, List<Tuple<Tuple<string, string>, int>>>()
                {
                    { "expected", new List<Tuple<Tuple<string, string>, int>>()
                        {
                            new Tuple<Tuple<string, string>, int>(Tuple.Create("feature_c", "c_val_1"), 1),
                            new Tuple<Tuple<string, string>, int>(Tuple.Create("feature_c", "c_val_2"), 1)
                        }
                    }
                }
            }
        };

        foreach (var entry in table)
        {
            string feature = entry.Key;
            var test = entry.Value;
            var counts = GetCounts(feature); // Assuming GetCounts is a method you define
            // You may need to adapt the comparison logic based on your needs
            // This is just a basic placeholder
            // Note: assertCountEqual doesn't directly translate, so adjust accordingly
            // this.AssertCountEqual(counts, test["expected"], feature);
        }

        // Test joint_counts
        Dictionary<Tuple<Tuple<string, string>, Tuple<string, string>>, int> expectedJointCounts = new Dictionary<Tuple<Tuple<string, string>, Tuple<string, string>>, int>()
        {
            { Tuple.Create(Tuple.Create("feature_a", "a_val_1"), Tuple.Create("feature_b", "b_val_1")), 2 },
            { Tuple.Create(Tuple.Create("feature_a", "a_val_1"), Tuple.Create("feature_c", "c_val_1")), 1 },
            { Tuple.Create(Tuple.Create("feature_a", "a_val_1"), Tuple.Create("feature_c", "c_val_2")), 1 },
            { Tuple.Create(Tuple.Create("feature_b", "b_val_1"), Tuple.Create("feature_c", "c_val_1")), 1 },
            { Tuple.Create(Tuple.Create("feature_b", "b_val_1"), Tuple.Create("feature_c", "c_val_2")), 1 },
        };

        // You may need to adapt the comparison logic based on your needs
        // This is just a basic placeholder
        // Note: assertDictEqual doesn't directly translate, so adjust accordingly
        // this.AssertDictEqual(this.jointCounts, expectedJointCounts);
    }

    // Your existing methods here...

    // Define your other methods as needed...
}
```
---
    
> **Indresh** - _(09/11/2023 10:20:30)_
```
using System;
using System.Collections.Generic;

class Program
{
    static void Main()
    {
        List<Dictionary<string, string>> data = new List<Dictionary<string, string>>
        {
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "master" }, { "Marriage", "divorced" }, { "Income", "low" } },
            new Dictionary<string, string> { { "Gender", "female" }, { "Education", "master" }, { "Marriage", "married" }, { "Income", "medium" } },
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "master" }, { "Marriage", "single" }, { "Income", "high" } },
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "bachelor" }, { "Marriage", "married" }, { "Income", "medium" } },
            new Dictionary<string, string> { { "Gender", "female" }, { "Education", "master" }, { "Marriage", "divorced" }, { "Income", "high" } },
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "PhD" }, { "Marriage", "married" }, { "Income", "high" } },
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "master" }, { "Marriage", "single" }, { "Income", "high" } },
            new Dictionary<string, string> { { "Gender", "female" }, { "Education", "PhD" }, { "Marriage", "single" }, { "Income", "medium" } },
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "PhD" }, { "Marriage", "married" }, { "Income", "medium" } },
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "bachelor" }, { "Marriage", "single" }, { "Income", "low" } },
            new Dictionary<string, string> { { "Gender", "female" }, { "Education", "PhD" }, { "Marriage", "married" }, { "Income", "medium" } },
            new Dictionary<string, string> { { "Gender", "male" }, { "Education", "master" }, { "Marriage", "single" }, { "Income", "low" } }
        };

        // Accessing the data:
        Console.WriteLine("Gender of the first entry: " + data[0]["Gender"]);
        Console.WriteLine("Education of the second entry: " + data[1]["Education"]);
        // Access other fields as needed...

        // You can loop through the list and dictionaries to process the data.
    }
}
```
---
    
> **Indresh** - _(09/11/2023 16:18:42)_
```
using System;
using System.Collections.Generic;

class Detector
{
    public List<double> Score(List<Dictionary<string, string>> observations)
    {
        // Implementation of the score method
        // Replace this with the actual implementation
        List<double> scores = new List<double>();
        // ...

        return scores;
    }

    public List<Dictionary<string, double>> ValueScores(List<Dictionary<string, string>> observations)
    {
        // Implementation of the value_scores method
        // Replace this with the actual implementation
        List<Dictionary<string, double>> valueScores = new List<Dictionary<string, double>>();
        // ...

        return valueScores;
    }

    // Other members of the Detector class can be added here
}

class Program
{
    static void Main()
    {
        Detector detector = new Detector();
        List<Dictionary<string, string>> observations = new List<Dictionary<string, string>> { /* Add your observations here */ };

        List<double> scores = detector.Score(observations);
        List<Dictionary<string, double>> valueScores = detector.ValueScores(observations);

        // Display results
        Console.WriteLine($"Detector fit with {observations.Count} observations:");
        for (int i = 0; i < observations.Count; i++)
        {
            Console.WriteLine($"Observation ID {i + 1}: {DictionaryToString(observations[i])}");
        }

        Console.WriteLine("\nScores:");
        for (int i = 0; i < scores.Count; i++)
        {
            Console.WriteLine($"Observation ID {i + 1}: {Math.Round(scores[i], 4)}");
        }

        Console.WriteLine("\nValue scores per attribute:");
        for (int i = 0; i < valueScores.Count; i++)
        {
            Console.WriteLine($"Observation ID {i + 1}: {RoundDictionaryValues(valueScores[i], 4)}");
        }
    }

    static string DictionaryToString(Dictionary<string, string> dictionary)
    {
        // Convert dictionary to string for display
        // Replace this with the actual implementation
        // This is a simple example; you can customize it based on your needs
        return string.Join(", ", dictionary.Select(kv => $"{kv.Key}: {kv.Value}"));
    }

    static Dictionary<string, double> RoundDictionaryValues(Dictionary<string, double> dictionary, int decimalPlaces)
    {
        // Implement rounding for dictionary values
        // Replace this with the actual implementation
        Dictionary<string, double> roundedDictionary = new Dictionary<string, double>();
        foreach (var entry in dictionary)
        {
            roundedDictionary.Add(entry.Key, Math.Round(entry.Value, decimalPlaces));
        }

        return roundedDictionary;
    }
}
```
---
