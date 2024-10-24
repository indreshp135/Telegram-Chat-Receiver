```cs
using Microsoft.Graph;
using Microsoft.Identity.Client;
using Azure.Identity;
using System.Threading.Tasks;
using System.Collections.Generic;
using System.IO;
using CsvHelper;
using System.Globalization;
using System.Linq;
using System;

public class AzureAdOperations
{
    private readonly GraphServiceClient _graphClient;

    public AzureAdOperations(string tenantId, string clientId, string clientSecret)
    {
        var credentials = new ClientSecretCredential(
            tenantId,
            clientId,
            clientSecret);

        _graphClient = new GraphServiceClient(credentials);
    }

    // Get all users with all fields
    public async Task<List<User>> GetAllUsersAsync()
    {
        var users = new List<User>();
        var result = await _graphClient.Users
            .Request()
            .GetAsync();

        users.AddRange(result.CurrentPage);
        
        while (result.NextPageRequest != null)
        {
            result = await result.NextPageRequest.GetAsync();
            users.AddRange(result.CurrentPage);
        }

        return users;
    }

    // Get all roles with all fields
    public async Task<List<DirectoryRole>> GetAllRolesAsync()
    {
        var roles = new List<DirectoryRole>();
        var result = await _graphClient.DirectoryRoles
            .Request()
            .GetAsync();

        roles.AddRange(result.CurrentPage);

        while (result.NextPageRequest != null)
        {
            result = await result.NextPageRequest.GetAsync();
            roles.AddRange(result.CurrentPage);
        }

        return roles;
    }

    // Get all groups with all fields
    public async Task<List<Group>> GetAllGroupsAsync()
    {
        var groups = new List<Group>();
        var result = await _graphClient.Groups
            .Request()
            .GetAsync();

        groups.AddRange(result.CurrentPage);

        while (result.NextPageRequest != null)
        {
            result = await result.NextPageRequest.GetAsync();
            groups.AddRange(result.CurrentPage);
        }

        return groups;
    }

    // Get roles assigned to a group
    public async Task<List<DirectoryRole>> GetRolesForGroupAsync(string groupId)
    {
        var roles = new List<DirectoryRole>();
        var result = await _graphClient.Groups[groupId]
            .TransitiveMembers
            .Request()
            .GetAsync();

        foreach (var member in result.CurrentPage)
        {
            if (member is DirectoryRole role)
            {
                roles.Add(role);
            }
        }

        while (result.NextPageRequest != null)
        {
            result = await result.NextPageRequest.GetAsync();
            foreach (var member in result.CurrentPage)
            {
                if (member is DirectoryRole role)
                {
                    roles.Add(role);
                }
            }
        }

        return roles;
    }

    // Get users in a group
    public async Task<List<User>> GetUsersInGroupAsync(string groupId)
    {
        var users = new List<User>();
        var result = await _graphClient.Groups[groupId]
            .TransitiveMembers
            .Request()
            .GetAsync();

        foreach (var member in result.CurrentPage)
        {
            if (member is User user)
            {
                users.Add(user);
            }
        }

        while (result.NextPageRequest != null)
        {
            result = await result.NextPageRequest.GetAsync();
            foreach (var member in result.CurrentPage)
            {
                if (member is User user)
                {
                    users.Add(user);
                }
            }
        }

        return users;
    }

    // Get roles assigned to a user
    public async Task<List<DirectoryRole>> GetRolesForUserAsync(string userId)
    {
        var roles = new List<DirectoryRole>();
        var result = await _graphClient.Users[userId]
            .MemberOf
            .Request()
            .GetAsync();

        foreach (var member in result.CurrentPage)
        {
            if (member is DirectoryRole role)
            {
                roles.Add(role);
            }
        }

        while (result.NextPageRequest != null)
        {
            result = await result.NextPageRequest.GetAsync();
            foreach (var member in result.CurrentPage)
            {
                if (member is DirectoryRole role)
                {
                    roles.Add(role);
                }
            }
        }

        return roles;
    }

    // Batch operation to get roles for all groups
    public async Task<Dictionary<string, List<DirectoryRole>>> GetRolesForAllGroupsAsync()
    {
        var result = new Dictionary<string, List<DirectoryRole>>();
        var groups = await GetAllGroupsAsync();

        foreach (var group in groups)
        {
            try
            {
                var roles = await GetRolesForGroupAsync(group.Id);
                result.Add(group.Id, roles);
                Console.WriteLine($"Processed group: {group.DisplayName}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error processing group {group.DisplayName}: {ex.Message}");
            }
            // Add a small delay to avoid throttling
            await Task.Delay(100);
        }

        return result;
    }

    // Batch operation to get users for all groups
    public async Task<Dictionary<string, List<User>>> GetUsersForAllGroupsAsync()
    {
        var result = new Dictionary<string, List<User>>();
        var groups = await GetAllGroupsAsync();

        foreach (var group in groups)
        {
            try
            {
                var users = await GetUsersInGroupAsync(group.Id);
                result.Add(group.Id, users);
                Console.WriteLine($"Processed group: {group.DisplayName}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error processing group {group.DisplayName}: {ex.Message}");
            }
            await Task.Delay(100);
        }

        return result;
    }

    // Batch operation to get roles for all users
    public async Task<Dictionary<string, List<DirectoryRole>>> GetRolesForAllUsersAsync()
    {
        var result = new Dictionary<string, List<DirectoryRole>>();
        var users = await GetAllUsersAsync();

        foreach (var user in users)
        {
            try
            {
                var roles = await GetRolesForUserAsync(user.Id);
                result.Add(user.Id, roles);
                Console.WriteLine($"Processed user: {user.DisplayName}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error processing user {user.DisplayName}: {ex.Message}");
            }
            await Task.Delay(100);
        }

        return result;
    }

    // Export functions
    public async Task ExportUsersToCSV(string filePath)
    {
        var users = await GetAllUsersAsync();
        using (var writer = new StreamWriter(filePath))
        using (var csv = new CsvWriter(writer, CultureInfo.InvariantCulture))
        {
            csv.WriteRecords(users.Select(u => new {
                Id = u.Id,
                DisplayName = u.DisplayName,
                UserPrincipalName = u.UserPrincipalName,
                Mail = u.Mail,
                JobTitle = u.JobTitle,
                Department = u.Department,
                OfficeLocation = u.OfficeLocation,
                BusinessPhones = string.Join(";", u.BusinessPhones ?? new string[] { }),
                MobilePhone = u.MobilePhone
            }));
        }
    }

    public async Task ExportRolesToCSV(string filePath)
    {
        var roles = await GetAllRolesAsync();
        using (var writer = new StreamWriter(filePath))
        using (var csv = new CsvWriter(writer, CultureInfo.InvariantCulture))
        {
            csv.WriteRecords(roles.Select(r => new {
                Id = r.Id,
                DisplayName = r.DisplayName,
                Description = r.Description
            }));
        }
    }

    public async Task ExportGroupsToCSV(string filePath)
    {
        var groups = await GetAllGroupsAsync();
        using (var writer = new StreamWriter(filePath))
        using (var csv = new CsvWriter(writer, CultureInfo.InvariantCulture))
        {
            csv.WriteRecords(groups.Select(g => new {
                Id = g.Id,
                DisplayName = g.DisplayName,
                Description = g.Description,
                Mail = g.Mail,
                SecurityEnabled = g.SecurityEnabled,
                MailEnabled = g.MailEnabled
            }));
        }
    }

    public async Task ExportGroupMembershipsToCSV(string filePath)
    {
        var allGroupUsers = await GetUsersForAllGroupsAsync();
        var records = new List<dynamic>();

        foreach (var groupUsers in allGroupUsers)
        {
            var group = (await _graphClient.Groups[groupUsers.Key].Request().GetAsync());
            foreach (var user in groupUsers.Value)
            {
                records.Add(new {
                    GroupId = group.Id,
                    GroupName = group.DisplayName,
                    UserId = user.Id,
                    UserDisplayName = user.DisplayName,
                    UserPrincipalName = user.UserPrincipalName
                });
            }
        }

        using (var writer = new StreamWriter(filePath))
        using (var csv = new CsvWriter(writer, CultureInfo.InvariantCulture))
        {
            csv.WriteRecords(records);
        }
    }

    public async Task ExportUserRolesToCSV(string filePath)
    {
        var allUserRoles = await GetRolesForAllUsersAsync();
        var records = new List<dynamic>();

        foreach (var userRoles in allUserRoles)
        {
            var user = (await _graphClient.Users[userRoles.Key].Request().GetAsync());
            foreach (var role in userRoles.Value)
            {
                records.Add(new {
                    UserId = user.Id,
                    UserDisplayName = user.DisplayName,
                    UserPrincipalName = user.UserPrincipalName,
                    RoleId = role.Id,
                    RoleDisplayName = role.DisplayName
                });
            }
        }

        using (var writer = new StreamWriter(filePath))
        using (var csv = new CsvWriter(writer, CultureInfo.InvariantCulture))
        {
            csv.WriteRecords(records);
        }
    }
}
```
