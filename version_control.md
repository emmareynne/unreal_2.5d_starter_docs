# Git LFS Version Control for Unreal Engine

## Prerequisites
- Completed steps in project_setup.md
- [Git](https://git-scm.com/downloads) installed and configured
- [Git LFS](https://git-lfs.github.com/) installed and configured
- Basic understanding of version control concepts

## Git LFS Setup for Unreal Engine Projects

### Repository Configuration

1. **Repository Setup**:
   - Create a free repository on [GitHub](https://github.com/), [GitLab](https://gitlab.com/), or [Azure DevOps](https://azure.microsoft.com/en-us/products/devops/)
   - GitHub Free: 1GB LFS storage, unlimited collaborators for public repos, 3 for private
   - GitLab Free: 5GB total storage including LFS
   - Azure DevOps: 2GB Git storage per repo, 5 free users

2. **Initialize Git LFS**:
   ```bash
   git lfs install
   ```

3. **Configure Git LFS tracking** for Unreal Engine:
   ```bash
   git lfs track "*.uasset"
   git lfs track "*.umap"
   git lfs track "*.fbx"
   git lfs track "*.png"
   git lfs track "*.jpg"
   git lfs track "*.wav"
   git lfs track "*.mp3"
   git lfs track "*.mp4"
   ```

4. **Create a `.gitignore` file** for Unreal-specific files:
   ```
   # Unreal Engine
   Saved/
   Intermediate/
   DerivedDataCache/
   Build/
   Binaries/
   *.pdb
   *.vspscc
   
   # IDE files
   .vs/
   .vscode/
   *.sln
   *.suo
   
   # OS files
   .DS_Store
   Thumbs.db
   ```

### Repository Setup

1. **Initialize the Repository**:
   ```bash
   git init
   git add .gitattributes .gitignore
   git commit -m "Initial repository setup"
   ```

2. **Connect to Remote Repository**:
   ```bash
   git remote add origin https://github.com/username/your-game-name.git
   git push -u origin main
   ```

3. **Initial Commit**:
   ```bash
   cd [ProjectDirectory]
   git add .
   git commit -m "Initial project setup"
   git push
   ```

### Unreal Engine Integration

1. **Configure Unreal Editor**:
   - Go to Edit → Project Settings → Source Control
   - Select Git as provider (or Git LFS if available)
   - Enter your working directory path
   - Test the connection

2. **Enable File Locking with Git LFS**:
   - Add the following to .gitattributes to enable file locking:
   ```
   *.uasset filter=lfs diff=lfs merge=lfs -text lockable
   *.umap filter=lfs diff=lfs merge=lfs -text lockable
   ```
   - This is critical for Unreal binary assets to prevent conflicts

## Recommended Branching Strategy

Git Flow is recommended for Unreal Engine projects:

1. **Main Branch** (stable, production-ready code):
   ```bash
   git checkout -b main
   ```

2. **Development Branch** (integration branch):
   ```bash
   git checkout -b develop
   ```

3. **Feature Branches** as needed:
   ```bash
   git checkout -b feature/feature-name develop
   ```

4. **Release Branches** when preparing releases:
   ```bash
   git checkout -b release/v1.0 develop
   ```

## Git LFS Workflow Best Practices

### Working with Files

1. **Pulling Latest Changes**:
   - Command line: `git pull`
   - Always pull before starting new work

2. **Locking Binary Files** (if using lockable files):
   - Command line: `git lfs lock path/to/file.uasset`
   - Prevents others from modifying the same file

3. **Unlocking Files** when done:
   - Command line: `git lfs unlock path/to/file.uasset`
   - Always unlock files when you're done editing them

4. **Adding New Files**:
   - In Unreal Editor: New files are detected automatically
   - Command line: `git add [filename]`

5. **Committing Changes**:
   - Command line: `git commit -m "Description of changes"`

6. **Pushing Changes**:
   - Command line: `git push`

7. **Resolving Conflicts**:
   - For text conflicts: Standard Git conflict resolution
   - For binary conflicts: Use file locking to avoid them
   - For complex conflicts: Consider tools like [Beyond Compare](https://www.scootersoftware.com/)

### Branch Management

1. **Creating Feature Branches**:
   ```bash
   git checkout -b feature/my-feature develop
   ```

2. **Merging Feature Branches**:
   ```bash
   git checkout develop
   git merge --no-ff feature/my-feature
   git push
   ```

3. **Creating Release Branches**:
   ```bash
   git checkout -b release/v1.0 develop
   ```

4. **Finishing Release Branches**:
   ```bash
   git checkout main
   git merge --no-ff release/v1.0
   git tag -a v1.0 -m "Version 1.0"
   git checkout develop
   git merge --no-ff release/v1.0
   git branch -d release/v1.0
   ```

### Commit Best Practices

1. **Commit Frequency**:
   - Commit when you reach a working state
   - Smaller, focused commits are preferable
   - At least daily commits

2. **Description Format**:
   ```
   [Type] Short summary (50 chars or less)

   Detailed explanation if necessary. Wrap at 72 chars.
   Include motivation for change and differences from previous behavior.

   Reviewers: @username
   Task: TASK-123
   ```

3. **Types of Changes**:
   - [Feature] - New functionality
   - [Fix] - Bug fixes
   - [Refactor] - Code changes that don't add features or fix bugs
   - [Perf] - Performance improvements
   - [Style] - Code style/formatting changes
   - [Asset] - Adding or modifying game assets

## Unreal-Specific Workflows

### Asset Management

1. **Organize assets into logical folders**:
   - Characters
   - Environments
   - UI
   - Gameplay
   - Materials
   - Audio

2. **Use prefix naming conventions**:
   - BP_ for Blueprints
   - M_ for Materials
   - SM_ for Static Meshes
   - SK_ for Skeletal Meshes
   - T_ for Textures
   - WBP_ for Widget Blueprints

3. **File Locking Workflow**:
   - Lock binary assets before editing
   - Commit and unlock promptly when done
   - Communicate with team when working on important assets

### Managing Large Binary Files

1. **Bandwidth and Storage Optimization**:
   - Be mindful of the free tier limits (1-5GB depending on service)
   - Consider referencing external assets when possible
   - Use Asset Packs judiciously

2. **Git LFS Maintenance**:
   ```bash
   git lfs prune
   ```
   - Remove locally cached LFS files that aren't needed

## Collaborative Workflows

### Pull Requests and Code Reviews

1. **Create Pull Requests**:
   - Push your feature branch to the remote repository
   - Create a pull request from the feature branch to develop

2. **Review Process**:
   - Assign reviewers
   - Discuss changes using comments
   - Request updates if needed
   - Approve and merge when ready

### Handling Large Projects

1. **Component-Based Development**:
   - Split large projects into components
   - Consider Git submodules for shared components
   - Use sparse checkout for working on specific parts

2. **Optimizing Performance**:
   - Use shallow clones for faster initial downloads
   - Consider Git LFS partial clones
   - Use bfg-repo-cleaner for reducing repository size if needed

## Backup and Recovery

1. **Regular Backups**:
   - Set up repository mirroring to another service
   - Consider Git Bundle files for offline backups

2. **Disaster Recovery Plan**:
   - Document recovery procedures
   - Test the restoration process periodically

## Engine Version Migration

1. **Create a New Branch** for engine upgrades:
   ```bash
   git checkout -b engine-upgrade/5.4 develop
   ```

2. **Test thoroughly** after migration
3. **Document** any issues or workarounds
4. **Merge back** to develop only after successful testing

## Free Tier Management Tips

1. **Monitor LFS Storage Usage**:
   - GitHub: Check repository settings → Git LFS usage
   - GitLab: Project settings → Usage Quotas
   - Azure DevOps: Project settings → Storage

2. **Optimize Repository Size**:
   - Consider texture compression before committing
   - Use Git LFS garbage collection: `git lfs prune`
   - Archive older versions of large assets

3. **Exceed Free Tier Solutions**:
   - GitHub: $5/month for 50GB LFS storage if needed
   - GitLab: Premium tiers offer more storage
   - Consider [GitHub LFS migration to self-hosted](https://github.com/git-lfs/git-lfs/wiki/Tutorial:-Migrating-Existing-Repository-Data-To-Git-LFS#migrate-lfs-data-out-of-github) if approaching limits

## Next Steps
After setting up your Git LFS workflow, proceed with implementing the core framework outlined in `core_framework.md`. 