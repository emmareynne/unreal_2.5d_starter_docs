# Git LFS Version Control for Unreal Engine
(temporarily deprecated.. need to wait for project borealis to be updated, but ideally a gitlfs + project borealis implementation is best for this)

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
   Add to .gitattributes that was created after lfs install.

   ```bash
   # Unreal Engine file types.
   *.uasset filter=lfs diff=lfs merge=lfs -text
   *.umap filter=lfs diff=lfs merge=lfs -text
   *.uexp filter=lfs diff=lfs merge=lfs -text
   *.ubulk filter=lfs diff=lfs merge=lfs -text

   # Raw Content file types.
   *.fbx filter=lfs diff=lfs merge=lfs -text
   *.3ds filter=lfs diff=lfs merge=lfs -text
   *.psd filter=lfs diff=lfs merge=lfs -text
   *.png filter=lfs diff=lfs merge=lfs -text
   *.mp3 filter=lfs diff=lfs merge=lfs -text
   *.wav filter=lfs diff=lfs merge=lfs -text
   *.xcf filter=lfs diff=lfs merge=lfs -text
   *.jpg filter=lfs diff=lfs merge=lfs -text
   *.jpeg filter=lfs diff=lfs merge=lfs -text
   *.gif filter=lfs diff=lfs merge=lfs -text
   *.bmp filter=lfs diff=lfs merge=lfs -text
   *.tga filter=lfs diff=lfs merge=lfs -text
   *.tiff filter=lfs diff=lfs merge=lfs -text

   # Anything in `/RawContent` dir.
   /RawContent/**/* filter=lfs diff=lfs merge=lfs -text
   ```

4. **Create a `.gitignore` file** for Unreal-specific files:
   ```
   # Unreal Generated Files
   Saved/
   Intermediate/
   DerivedDataCache/
   Build/
   Binaries/

   # Visual Studio Files
   .vs/
   .vscode/
   *.sln
   *.suo
   *.opensdf
   *.sdf
   *.VC.db
   *.VC.opendb
   *.vspscc
   *.vsp
   *.vspx
   *.user
   ipch/
   
   # Compiled source files 
   *.pdb
   *.obj
   
   # OS generated files
   .DS_Store
   .DS_Store?
   ._*
   .Spotlight-V100
   .Trashes
   Icon?
   ehthumbs.db
   Thumbs.db
   
   # Builds
   *.apk
   *.ipa
   ```

### Repository Setup (Project Borealis Style)

1. **Initialize the Repository**:
   ```bash
   git init
   git add .gitattributes .gitignore
   git commit -m "Initial repository setup"
   ```

2. **Connect to Remote Repository**:
   ```bash
   git remote add origin https://github.com/username/your-game-name.git
   git branch -M main
   git push -u origin main
   ```

3. **Initial Commit**:
   ```bash
   # Add your project files
   git add .
   git commit -m "Initial project setup"
   git push
   ```

### Unreal Engine Integration

1. **Configure Unreal Editor**:
   - Go to Edit → Project Settings → Source Control
   - Set Source Control Login to:
     - Provider: Git (if available) or "None" (use Git outside the editor)
     - Username: Your Git username
     - Email: Your Git email
   - Enter your working directory path
   - Click "Accept Settings"

2. **File Locking Strategy** (Project Borealis approach):
   - Project Borealis doesn't use Git LFS file locking
   - Instead, they rely on team communication and coordination
   - For larger teams, consider establishing a check-out protocol via communication channels
   - For solo developers, locking isn't necessary

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

2. **Adding New Files**:
   - In Unreal Editor: New files are detected automatically
   - Command line: `git add [filename]`
   - For new binary file types, ensure they're tracked in .gitattributes

3. **Committing Changes**:
   - Command line: `git commit -m "Description of changes"`
   - Follow Project Borealis commit message format:
     ```
     [Type] Description of change
     ```

4. **Pushing Changes**:
   - Command line: `git push`
   - If bandwidth is an issue, consider using `git push --no-verify`

5. **Resolving Conflicts**:
   - For text conflicts: Standard Git conflict resolution
   - For binary conflicts: Choose which version to keep or merge manually
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

2. **Description Format** (Project Borealis style):
   ```
   [Type] Short summary
   ```
   
   Where Type is one of:
   - Feature - New functionality
   - Fix - Bug fixes
   - Refactor - Code changes that don't add features or fix bugs
   - Perf - Performance improvements
   - Style - Code style/formatting changes
   - Asset - Adding or modifying game assets
   - Doc - Documentation changes

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

3. **Binary Asset Workflow**:
   - Communicate with team members before making large changes
   - Consider using a "RawContent" folder for original art assets (PSD, etc.)
   - Use Unreal's Asset Browser to track what you're working on

### Managing Large Binary Files

1. **Bandwidth and Storage Optimization**:
   - Be mindful of the free tier limits (1-5GB depending on service)
   - Consider referencing external assets when possible
   - Use Asset Packs judiciously
   - Consider texture compression before committing

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
   - Include before/after screenshots for visual changes

2. **Review Process**:
   - Assign reviewers
   - Discuss changes using comments
   - Request updates if needed
   - Approve and merge when ready

### Project Borealis Collaboration Tips

1. **Communication First**:
   - Announce major asset changes in team chat
   - Discuss large refactors before implementing
   - Document design decisions

2. **Handling Large Projects**:
   - Split large projects into components
   - Consider Git submodules for shared components
   - Use shallow clones for faster initial downloads

3. **Performance Optimization**:
   - Use shallow clones for faster initial downloads
   - Consider Git LFS partial clones
   - Use `git lfs fetch --recent` to get only recent assets

## Backup and Recovery

1. **Regular Backups**:
   - Set up repository mirroring to another service
   - Consider Git Bundle files for offline backups
   - Use `git clone --mirror` for full backups including refs

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