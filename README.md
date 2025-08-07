```C#
using Google.Apis.Auth.OAuth2;
using Google.Cloud.ResourceManager.V3;
using Google.Cloud.Iam.V1;
using System;
using System.Threading.Tasks;
using System.Collections.Generic;

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
        Console.WriteLine("-------------------------------------------------------");

        await GetAndPrintAllIamPolicies(organizationId);

        Console.WriteLine("-------------------------------------------------------");
        Console.WriteLine("Scraping complete.");
    }

    public static async Task GetAndPrintAllIamPolicies(string organizationId)
    {
        // Get Organization IAM Policy
        await PrintOrganizationPolicy(organizationId);

        // Traverse all folders and projects under the organization
        await TraverseHierarchy($"organizations/{organizationId}");
    }

    private static async Task TraverseHierarchy(string parentName)
    {
        // Traverse folders
        var folders = _foldersClient.ListFoldersAsync(new ListFoldersRequest { Parent = parentName });
        await foreach (var folder in folders)
        {
            await PrintFolderPolicy(folder.Name);
            // Recursively traverse subfolders and projects
            await TraverseHierarchy(folder.Name);
        }

        // Traverse projects
        string filter = $"parent:{parentName}";
        var projects = _projectsClient.ListProjectsAsync(new ListProjectsRequest { Filter = filter });
        await foreach (var project in projects)
        {
            await PrintProjectPolicy(project.Name);
            // Get service account policies for the project
            await PrintServiceAccountPolicies(project.ProjectId);
        }
    }

    private static async Task PrintOrganizationPolicy(string organizationId)
    {
        try
        {
            var policy = await _organizationsClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = $"organizations/{organizationId}" });
            Console.WriteLine($"\n--- IAM Policy for Organization: organizations/{organizationId} ---");
            PrintPolicy(policy);
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting organization policy for {organizationId}: {e.Message}");
        }
    }

    private static async Task PrintFolderPolicy(string folderName)
    {
        try
        {
            var policy = await _foldersClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = folderName });
            Console.WriteLine($"\n--- IAM Policy for Folder: {folderName} ---");
            PrintPolicy(policy);
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting folder policy for {folderName}: {e.Message}");
        }
    }

    private static async Task PrintProjectPolicy(string projectName)
    {
        try
        {
            var policy = await _projectsClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = projectName });
            Console.WriteLine($"\n--- IAM Policy for Project: {projectName} ---");
            PrintPolicy(policy);
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting project policy for {projectName}: {e.Message}");
        }
    }

    private static async Task PrintServiceAccountPolicies(string projectId)
    {
        try
        {
            var accounts = _iamClient.ListServiceAccountsAsync($"projects/{projectId}");
            await foreach (var account in accounts)
            {
                var policy = await _iamClient.GetIamPolicyAsync(new GetIamPolicyRequest { Resource = account.Name });
                Console.WriteLine($"\n--- IAM Policy for Service Account: {account.Email} in project {projectId} ---");
                PrintPolicy(policy);
            }
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting service account policies in project {projectId}: {e.Message}");
        }
    }

    private static void PrintPolicy(Google.Cloud.Iam.V1.Policy policy)
    {
        if (policy.Bindings.Count == 0)
        {
            Console.WriteLine("No IAM policy bindings found.");
            return;
        }

        foreach (var binding in policy.Bindings)
        {
            Console.WriteLine($"  Role: {binding.Role}");
            Console.WriteLine("  Members:");
            foreach (var member in binding.Members)
            {
                Console.WriteLine($"    - {member}");
            }
        }
    }
}
```
