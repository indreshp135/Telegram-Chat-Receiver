
> **Indresh** - _(02/11/2023 05:40:48)_
```
import os
import git
import re

# List of repository URLs
repos = ["https://github.com/repo1.git", "https://github.com/repo2.git"]

# New branch name
new_branch_name = "update_vs_version"

# Iterate through the repositories
for repo_url in repos:
    # Clone the repository
    repo_name = os.path.splitext(os.path.basename(repo_url))[0]
    git.Repo.clone_from(repo_url, repo_name)

    # Change directory to the cloned repository
    os.chdir(repo_name)

    # Create a new branch
    repo = git.Repo('.')
    repo.git.checkout('-b', new_branch_name)

    # Find .sln file and update VisualStudioVersion
    sln_file = None
    for root, dirs, files in os.walk('.'):
        for file in files:
            if file.endswith('.sln'):
                sln_file = os.path.join(root, file)
                break

    if sln_file:
        with open(sln_file, 'r') as file:
            sln_contents = file.read()

        # Update VisualStudioVersion to 17
        sln_contents = re.sub(r'VisualStudioVersion = \d+', 'VisualStudioVersion = 17', sln_contents)

        with open(sln_file, 'w') as file:
            file.write(sln_contents)

        # Commit and push the changes
        repo.git.add(sln_file)
        repo.index.commit("Update VisualStudioVersion to 17")
        origin = repo.remote(name='origin')
        origin.push(new_branch_name)

    # Change directory back to the parent directory for the next repository
    os.chdir('..')
```
---
    
> **Indresh** - _(02/11/2023 05:59:08)_
```
const nodegit = require('nodegit');
const fs = require('fs').promises;

// List of repository URLs
const repos = ["https://your_pat@github.com/repo1.git", "https://your_pat@github.com/repo2.git"];

// New branch name
const newBranchName = "update_vs_version";

async function cloneAndUpdateRepos() {
  for (const repoUrl of repos) {
    // Clone the repository
    const repoName = repoUrl.split('/').pop().replace('.git', '');
    const repoPath = ./${repoName};
    const repo = await nodegit.Clone(repoUrl, repoPath, {
      fetchOpts: {
        callbacks: {
          credentials: (_url, username) => nodegit.Cred.userpassPlaintextNew(username, 'x-oauth-basic'),
        },
      },
    });

    // Create a new branch
    const branch = await repo.createBranch(newBranchName);
    await repo.checkoutBranch(branch);

    // Find .sln file and update VisualStudioVersion
    const slnFile = await findSlnFile(repoPath);
    if (slnFile) {
      const slnContent = await fs.readFile(slnFile, 'utf8');
      const updatedContent = slnContent.replace(/VisualStudioVersion = \d+/, 'VisualStudioVersion = 17');
      await fs.writeFile(slnFile, updatedContent);

      // Commit the changes
      const index = await repo.refreshIndex();
      await index.addByPath(slnFile);
      await index.write();
      const oid = await index.writeTree();
      const head = await nodegit.Reference.nameToId(repo, 'HEAD');
      const parent = await repo.getCommit(head);
      const signature = nodegit.Signature.now('Your Name', 'your.email@example.com');
      await repo.createCommit('HEAD', signature, signature, 'Update VisualStudioVersion', oid, [parent]);

      // Push the changes
      const remote = await repo.getRemote('origin');
      await remote.push([refs/heads/${newBranchName}:refs/heads/${newBranchName}]);
    }
  }
}

async function findSlnFile(directory) {
  const files = await fs.readdir(directory);
  for (const file of files) {
    if (file.endsWith('.sln')) {
      return ${directory}/${file};
    }
  }
  return null;
}

cloneAndUpdateRepos().catch(err => console.error(err));
```
---
    
> **Indresh** - _(03/11/2023 07:18:35)_
```
import git

# Replace 'repo_path' with the path to your Git repository
repo_path = '/path/to/your/repo'

# Create a Git repository object
repo = git.Repo(repo_path)

# Name of the branch you want to create
new_branch_name = 'develop'

# Check if 'develop' branch exists
if new_branch_name not in [branch.name for branch in repo.branches]:
    # Get the 'master' branch
    master_branch = repo.branches['master']

    # Create a new branch from 'master'
    new_branch = master_branch.checkout(b=new_branch_name)

    # Push the new branch to the remote repository
    new_branch.remote().push(new_branch.name)

    print(f"Created and pushed '{new_branch_name}' branch from 'master'.")
else:
    print(f"'{new_branch_name}' branch already exists.")
```
---
    
> **Indresh** - _(07/11/2023 04:59:27)_
```
import requests
import base64

# Set your GitHub repository and token details
repo_owner = "your_username"
repo_name = "your_repository"
branch_name = "main"
readme_file_path = "README.md"
access_token = "your_access_token"

# Fetch the contents of the README file
url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/contents/{readme_file_path}"
headers = {
    "Authorization": f"token {access_token}"
}
response = requests.get(url, headers=headers)
data = response.json()

if "content" in data:
    # Decode the base64 content
    readme_content = base64.b64decode(data["content"]).decode("utf-8")

    # Make changes to the README file content
    new_readme_content = readme_content + "\n\nUpdated content"

    # Create a new branch
    new_branch_name = "new-branch"
    branch_url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/git/refs/heads/{branch_name}"
    branch_data = {
        "ref": f"refs/heads/{new_branch_name}",
        "sha": data["sha"]
    }
    response = requests.post(branch_url, json=branch_data, headers=headers)

    if response.status_code == 201:
        # Commit the changes to the new branch
        commit_message = "Updated README.md"
        commit_url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/git/commits"
        commit_data = {
            "message": commit_message,
            "tree": data["sha"],
            "parents": [data["sha"]],
            "committer": {
                "name": "Your Name",
                "email": "youremail@example.com"
            },
            "author": {
                "name": "Your Name",
                "email": "youremail@example.com"
            },
            "signature": "-----"
        }
        response = requests.post(commit_url, json=commit_data, headers=headers)
        commit = response.json()

        # Update the branch reference to the new commit
        branch_ref_url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/git/refs/heads/{new_branch_name}"
        branch_ref_data = {
            "sha": commit["sha"]
        }
        response = requests.patch(branch_ref_url, json=branch_ref_data, headers=headers)

        if response.status_code == 200:
            # Create a pull request
            pr_url = f"https://api.github.com/repos/{repo_owner}/{repo_name}/pulls"
            pr_data = {
                "title": "Update README.md",
                "head": new_branch_name,
                "base": branch_name
            }
            response = requests.post(pr_url, json=pr_data, headers=headers)

            if response.status_code == 201:
                print("Pull request created successfully.")
            else:
                print("Error creating pull request.")
        else:
            print("Error updating branch reference.")
    else:
        print("Error creating new branch.")
else:
    print("Error fetching README file content.")
```
---
    