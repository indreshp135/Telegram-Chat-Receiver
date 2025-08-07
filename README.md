```C#
using Google.Apis.Auth.OAuth2;
using Google.Cloud.ResourceManager.V3;
using Google.Cloud.Iam.V1;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

public class PolicyScraper
{
    private readonly OrganizationsClient _organizationsClient;
    private readonly FoldersClient _foldersClient;
    private readonly ProjectsClient _projectsClient;
    private readonly IAMClient _iamClient;
    private readonly GoogleCredential _credential;

    public PolicyScraper()
    {
        _credential = GoogleCredential.GetApplicationDefault();
        _organizationsClient = new OrganizationsClientBuilder { Credential = _credential }.Build();
        _foldersClient = new FoldersClientBuilder { Credential = _credential }.Build();
        _projectsClient = new ProjectsClientBuilder { Credential = _credential }.Build();
        _iamClient = new IAMClientBuilder { Credential = _credential }.Build();
    }

    public async Task GetAndPrintAllIamPolicies(string organizationId)
    {
        await PrintOrganizationPolicy(organizationId);
        await TraverseFolders(organizationId);
    }

    private async Task TraverseFolders(string parentId, string parentType = "organizations")
    {
        string parentName = $"{parentType}/{parentId}";
        var folders = _foldersClient.ListFoldersAsync(parentName);

        await foreach (var folder in folders)
        {
            await PrintFolderPolicy(folder.Name);
            await TraverseFolders(folder.Name.Split('/')[1], "folders"); // Recurse for subfolders
            await TraverseProjects(folder.Name.Split('/')[1], "folders");
        }
        
        // This handles projects directly under the organization
        if (parentType == "organizations")
        {
            await TraverseProjects(parentId, parentType);
        }
    }

    private async Task TraverseProjects(string parentId, string parentType)
    {
        string filter = $"parent.id:{parentId} parent.type:{parentType}";
        var projects = _projectsClient.ListProjectsAsync(filter);
        
        await foreach (var project in projects)
        {
            await PrintProjectPolicy(project.Name);
            await PrintServiceAccountPolicies(project.ProjectId);
        }
    }

    private async Task PrintOrganizationPolicy(string organizationId)
    {
        try
        {
            var policy = await _organizationsClient.GetIamPolicyAsync(OrganizationName.FromOrganization(organizationId));
            Console.WriteLine($"\n--- IAM Policy for Organization: {organizationId} ---");
            PrintPolicy(policy);
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting organization policy for {organizationId}: {e.Message}");
        }
    }

    private async Task PrintFolderPolicy(string folderName)
    {
        try
        {
            var policy = await _foldersClient.GetIamPolicyAsync(folderName);
            Console.WriteLine($"\n--- IAM Policy for Folder: {folderName} ---");
            PrintPolicy(policy);
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting folder policy for {folderName}: {e.Message}");
        }
    }

    private async Task PrintProjectPolicy(string projectName)
    {
        try
        {
            var policy = await _projectsClient.GetIamPolicyAsync(projectName);
            Console.WriteLine($"\n--- IAM Policy for Project: {projectName} ---");
            PrintPolicy(policy);
        }
        catch (Exception e)
        {
            Console.WriteLine($"Error getting project policy for {projectName}: {e.Message}");
        }
    }

    private async Task PrintServiceAccountPolicies(string projectId)
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

    private void PrintPolicy(Google.Cloud.Iam.V1.Policy policy)
    {
        if (policy.Bindings.Count == 0)
        {
            Console.WriteLine("No IAM policy bindings found.");
            return;
        }

        foreach (var binding in policy.Bindings)
        {
            Console.WriteLine($"\n  Role: {binding.Role}");
            Console.WriteLine("  Members:");
            foreach (var member in binding.Members)
            {
                Console.WriteLine($"    - {member}");
            }
        }
    }
}

public class Program
{
    public static async Task Main(string[] args)
    {
        string organizationId = "YOUR_ORGANIZATION_ID";
        var scraper = new PolicyScraper();
        await scraper.GetAndPrintAllIamPolicies(organizationId);
    }
}
```
