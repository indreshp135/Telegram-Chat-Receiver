
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
    