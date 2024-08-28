```cs
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Threading.Tasks;
using System.IO;

public class SplunkQueryCsvExporter
{
    private readonly string _splunkHost;
    private readonly int _splunkPort;
    private readonly string _splunkUsername;
    private readonly string _splunkPassword;
    private readonly HttpClient _httpClient;

    public string Query { get; set; }
    public DateTime EarliestTime { get; set; }
    public DateTime LatestTime { get; set; }
    public string OutputDirectory { get; set; }

    public SplunkQueryCsvExporter(string splunkHost, int splunkPort, string splunkUsername, string splunkPassword,
                                  string query, DateTime earliestTime, DateTime latestTime, string outputDirectory)
    {
        _splunkHost = splunkHost;
        _splunkPort = splunkPort;
        _splunkUsername = splunkUsername;
        _splunkPassword = splunkPassword;
        Query = query;
        EarliestTime = earliestTime;
        LatestTime = latestTime;
        OutputDirectory = outputDirectory;

        _httpClient = new HttpClient();
        var authString = Convert.ToBase64String(Encoding.ASCII.GetBytes($"{_splunkUsername}:{_splunkPassword}"));
        _httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic", authString);
    }

    public async Task ExportToCsvAsync()
    {
        var csvContent = await ExecuteSearchAndGetCsvAsync();

        string fileName = $"splunk_results_{DateTime.Now:yyyyMMddHHmmss}.csv";
        string filePath = Path.Combine(OutputDirectory, fileName);

        await File.WriteAllTextAsync(filePath, csvContent);

        Console.WriteLine($"CSV file exported to: {filePath}");
    }

    private async Task<string> ExecuteSearchAndGetCsvAsync()
    {
        var earliest = EarliestTime.ToString("yyyy-MM-ddTHH:mm:ss");
        var latest = LatestTime.ToString("yyyy-MM-ddTHH:mm:ss");

        var content = new FormUrlEncodedContent(new[]
        {
            new KeyValuePair<string, string>("search", Query),
            new KeyValuePair<string, string>("earliest_time", earliest),
            new KeyValuePair<string, string>("latest_time", latest),
            new KeyValuePair<string, string>("output_mode", "csv")
        });

        var response = await _httpClient.PostAsync(
            $"https://{_splunkHost}:{_splunkPort}/services/search/v2/jobs/export",
            content);

        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
```
