```C#
using Google.Apis.Auth.OAuth2;
using Google.Cloud.ResourceManager.V3;
using Google.Cloud.Iam.V1;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using System.IO;

public class Program
{
    private static OrganizationsClient _organizationsClient;
    private static FoldersClient _foldersClient;
    private static ProjectsClient _projectsClient;
    private static IAMClient _iamClient;

    public static async Task Main(string[] args)
    {
        if (args.Length != 1)
        {
            Console.WriteLine("Usage: dotnet run <organizationId>");
            return;
        }

        string organizationId = args[0];

        // Authenticate with Google Cloud
        GoogleCredential credential = await GoogleCredential.GetApplicationDefaultAsync();
        _organizationsClient = new OrganizationsClientBuilder { Credential = credential }.Build();
        _foldersClient = new FoldersClientBuilder { Credential = credential }.Build();
        _projectsClient = new ProjectsClientBuilder { Credential = credential }.Build();
        _iamClient = new IAMClientBuilder { Credential = credential }.Build();

        Console.WriteLine($"Starting to scrape IAM policies for organization: {organizationId}");
        
        var allPolicies = await GetPoliciesForOrganization(organizationId);

        // Serialize the list of policies to a single JSON object
        string jsonOutput = JsonConvert.SerializeObject(allPolicies, Formatting.Indented);
        string outputFilePath = "iam_policies.json";
        await File.WriteAllTextAsync(outputFilePath, jsonOutput);

        Console.WriteLine($"\nScraping complete. All policies saved to {outputFilePath}");
    }

    private static async Task<List<PolicyInfo>> GetPoliciesForOrganization(string organizationId)
    {
        var policies = new List<PolicyInfo>();
        
        // Get Organization IAM Policy
        var orgPolicy = await GetOrganizationPolicy(organizationId);
        if (orgPolicy != null)
        {
            policies.Add(orgPolicy);
        }

        // Traverse all folders and projects under the organization
        await TraverseHierarchy($"organizations/{organizationId}", policies);

        return policies;
    }

    private static async Task TraverseHierarchy(string parentName, List<PolicyInfo> policies)
    {
        // Traverse folders
        var folders = _foldersClient.ListFoldersAsync(new ListFoldersRequest { Parent = parentName });
        await foreach (var folder in folders)
        {
            var folderPolicy = await GetFolderPolicy(folder.Name);
            if (folderPolicy != null)
            {
                policies.Add(folderPolicy);
            }
            // Recursively traverse subfolders and projects
            await TraverseHierarchy(folder.Name, policies);
        }

        // Traverse projects
        string filter = $"parent:{parentName}";
        var projects = _projectsClient.ListProjectsAsync(new ListProjectsRequest { Filter = filter });
        await foreach (var project in projects)
        {
            var projectPolicy = await GetProjectPolicy(project.Name);
            if (projectPolicy != null)
            {
                policies.Add(projectPolicy);
            }
            
            // Get service account policies for the project
            var saPolicies = await GetServiceAccountPolicies(project.ProjectId);
            policies.AddRange(saPolicies);
        }
    }

    private static async Task<PolicyInfo> GetOrganizationPolicy(string organizationId)
    {
        try
        {
            var policy = await _organizationsClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = $"organizations/{organizationId}" });
            return new PolicyInfo
            {
                ResourceType = "Organization",
                ResourceId = organizationId,
                Policy = policy
            };
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting organization policy for {organizationId}: {e.Message}");
            return null;
        }
    }

    private static async Task<PolicyInfo> GetFolderPolicy(string folderName)
    {
        try
        {
            var policy = await _foldersClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = folderName });
            return new PolicyInfo
            {
                ResourceType = "Folder",
                ResourceId = folderName,
                Policy = policy
            };
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting folder policy for {folderName}: {e.Message}");
            return null;
        }
    }

    private static async Task<PolicyInfo> GetProjectPolicy(string projectName)
    {
        try
        {
            var policy = await _projectsClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = projectName });
            return new PolicyInfo
            {
                ResourceType = "Project",
                ResourceId = projectName,
                Policy = policy
            };
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting project policy for {projectName}: {e.Message}");
            return null;
        }
    }

    private static async Task<List<PolicyInfo>> GetServiceAccountPolicies(string projectId)
    {
        var policies = new List<PolicyInfo>();
        try
        {
            var accounts = _iamClient.ListServiceAccountsAsync($"projects/{projectId}");
            await foreach (var account in accounts)
            {
                var policy = await _iamClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = account.Name });
                policies.Add(new PolicyInfo
                {
                    ResourceType = "ServiceAccount",
                    ResourceId = account.Email,
                    Policy = policy
                });
            }
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting service account policies in project {projectId}: {e.Message}");
        }
        return policies;
    }
}

public class PolicyInfo
{
    public string ResourceType { get; set; }
    public string ResourceId { get; set; }
    public Google.Cloud.Iam.V1.Policy Policy { get; set; }
}
```
