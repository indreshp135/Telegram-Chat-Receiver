public class CsvFieldValidator
{
    private readonly Dictionary<Type, Func<string, bool>> _typeValidators;
    private readonly Dictionary<Type, string> _typeDefaults;

    public CsvFieldValidator()
    {
        _typeValidators = new Dictionary<Type, Func<string, bool>>
        {
            { typeof(int), value => int.TryParse(value, out _) },
            { typeof(long), value => long.TryParse(value, out _) },
            { typeof(decimal), value => decimal.TryParse(value, out _) },
            { typeof(double), value => double.TryParse(value, out _) },
            { typeof(DateTime), value => DateTime.TryParse(value, out _) },
            { typeof(bool), value => bool.TryParse(value, out _) },
            { typeof(Guid), value => Guid.TryParse(value, out _) }
        };

        _typeDefaults = new Dictionary<Type, string>
        {
            { typeof(int), "0" },
            { typeof(long), "0" },
            { typeof(decimal), "0.0" },
            { typeof(double), "0.0" },
            { typeof(DateTime), DateTime.MinValue.ToString("yyyy-MM-dd") },
            { typeof(bool), "false" },
            { typeof(Guid), Guid.Empty.ToString() },
            { typeof(string), "" }
        };
    }

    public (bool IsValid, string ErrorMessage, string DefaultValue) ValidateField(string value, Type expectedType)
    {
        // Empty values should get default value for type
        if (string.IsNullOrWhiteSpace(value))
        {
            return (false, "Empty value", GetDefaultValue(expectedType));
        }

        // String type is always valid
        if (expectedType == typeof(string))
        {
            return (true, null, value);
        }

        // Check if we have a validator for this type
        if (_typeValidators.TryGetValue(expectedType, out var validator))
        {
            bool isValid = validator(value);
            if (!isValid)
            {
                return (false, 
                    $"Cannot parse '{value}' to type {expectedType.Name}", 
                    GetDefaultValue(expectedType));
            }
            return (true, null, value);
        }

        // If no validator found, return as is
        return (true, null, value);
    }

    private string GetDefaultValue(Type type)
    {
        return _typeDefaults.TryGetValue(type, out var defaultValue) 
            ? defaultValue 
            : string.Empty;
    }
}

public class CsvErrorHandler
{
    private readonly ILogger _logger;
    private readonly CsvConfiguration _csvConfig;
    private readonly CsvFieldValidator _validator;

    public CsvErrorHandler(ILogger logger)
    {
        _logger = logger;
        _validator = new CsvFieldValidator();
        _csvConfig = new CsvConfiguration(CultureInfo.InvariantCulture)
        {
            ReadingExceptionOccurred = context =>
            {
                var record = context.Exception.ReadingContext.RawRecord;
                var fields = record.Split(',').ToList();
                var hasErrors = false;
                var properties = typeof(SampleData).GetProperties();

                for (int i = 0; i < Math.Min(fields.Count, properties.Length); i++)
                {
                    var (isValid, errorMessage, defaultValue) = _validator.ValidateField(
                        fields[i], 
                        properties[i].PropertyType
                    );

                    if (!isValid)
                    {
                        hasErrors = true;
                        _logger.LogError($"Row {context.Exception.ReadingContext.RawRow}, Field {properties[i].Name}: {errorMessage}. Using default value: {defaultValue}");
                        fields[i] = defaultValue;
                    }
                }

                if (hasErrors)
                {
                    context.Exception.ReadingContext.RawRecord = string.Join(",", fields);
                    return true; // Continue with cleaned data
                }

                return false; // No errors found
            }
        };
    }

    public IEnumerable<T> ReadCsvFile<T>(string filePath)
    {
        using var reader = new StreamReader(filePath);
        using var csv = new CsvReader(reader, _csvConfig);
        
        var records = new List<T>();
        
        while (csv.Read())
        {
            try
            {
                var record = csv.GetRecord<T>();
                if (record != null)
                {
                    records.Add(record);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error processing record: {ex.Message}");
                // Continue to next record
            }
        }
        
        return records;
    }
}