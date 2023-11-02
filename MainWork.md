
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
    