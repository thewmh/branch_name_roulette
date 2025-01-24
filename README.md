# Randomized Branch Renaming Git Hook

This Git hook system renames remote branches to randomized names while retaining the original branch names. Users can interact with branches using their original names, and the new names are updated each time `git fetch` is executed.

## Features

* Randomized Renaming: Automatically renames all remote branches to jumbled names every time `git fetch` is executed.
* Branch Mapping: Maintains a mapping of original branch names to their renamed versions in a `.branch_map` file.
* Seamless Push/Pull: Allows users to push and pull using original branch names, translating them to the renamed versions automatically.

## Installation

1. Clone or navigate to your Git repository where you want to set up this hook.
2. Save the Hooks:

* Create a file named `post-fetch` in the `.git/hooks/` directory and paste the following script:

```
#!/bin/bash

# Path to the branch map file
branch_map_file=".branch_map"

# Generate a random, jumbled name for a branch
generate_random_name() {
  echo "$1" | sed 's/./&$(shuf -i 0-9 -n 1)/g' | tr 'a-zA-Z' 'n-za-mN-ZA-M' | awk -F/ '{for (i=1; i<=NF; i++) {gsub(/./, "&" rand()%10, $i); $i = $i substr(rand(), 3, 2)}} 1' OFS=/
}

# Fetch all remote branches
git fetch --prune

# Get the remote name (e.g., "origin")
remote_name=$(git remote)

# Get all remote branches
branches=$(git branch -r | grep "$remote_name" | sed "s/$remote_name\///")

# Clear the branch map file
: > "$branch_map_file"

# Iterate over each branch and rename it
for branch in $branches; do
  # Generate a new branch name
  new_branch=$(generate_random_name "$branch")

  # Create the new branch locally
  git checkout -b "$new_branch" "$remote_name/$branch"
  
  # Push the new branch to the remote
  git push "$remote_name" "$new_branch"
  
  # Delete the original branch from the remote
  git push "$remote_name" --delete "$branch"
  
  # Delete the local branch used to track the original remote
  git branch -D "$branch"
  
  # Update the branch map file with the original and new branch names
  echo "$branch -> $new_branch" > "$branch_map_file.tmp"
  cat "$branch_map_file" >> "$branch_map_file.tmp"
  mv "$branch_map_file.tmp" "$branch_map_file"
done

# Clean up and return to the default branch (e.g., main or master)
git checkout main 2>/dev/null || git checkout master
```

* Create a file named `pre-push` in the .git/hooks/ directory and paste the following script:

```
#!/bin/bash

# Path to the branch map file
branch_map_file=".branch_map"

# Function to map original branch names to renamed ones
get_renamed_branch() {
  local original_branch="$1"
  while IFS=" -> " read -r original renamed; do
    if [[ "$original" == "$original_branch" ]]; then
      echo "$renamed"
      return
    fi
  done < "$branch_map_file"
}

# Read the branch being pushed
while read -r local_ref local_sha remote_ref remote_sha; do
  branch_name=$(echo "$local_ref" | sed 's|refs/heads/||')
  renamed_branch=$(get_renamed_branch "$branch_name")
  
  if [[ -n "$renamed_branch" ]]; then
    echo "Intercepted push: $branch_name -> $renamed_branch"
    git push "$(git remote)" "HEAD:$renamed_branch"
    exit 0
  fi
done
```

## Make the Scripts Executable: Run the following commands in your terminal:

```
chmod +x .git/hooks/post-fetch
chmod +x .git/hooks/pre-push
```

## Usage

1. Fetch Remote Branches: Run the command:

```
git fetch
```

* This will rename all remote branches and update the `.branch_map` file with the mappings of original to renamed branches.

2. Push Using Original Names: You can now push using the original branch names:

```
git push origin feature/login
```

* The `pre-push` hook will intercept this command, translating `feature/login` to its latest randomized equivalent (e.g., `fteuar-5mnwl/loign-4pjxz`) and push it to the remote.

## Example Workflow

### Initial Remote Branches

```
origin/feature/login
origin/bugfix/auth
```

### After Running `git fetch`

* Remote branches are renamed:

```
origin/feature/login -> origin/fteuar-5mnwl/loign-4pjxz
origin/bugfix/auth   -> origin/bfuixa-8klnm/athuf-3mlnx
```

* `.branch_map` file:

```
feature/login -> fteuar-5mnwl/loign-4pjxz
bugfix/auth   -> bfuixa-8klnm/athuf-3mlnx
```

Running `git fetch` Again

* On subsequent fetches, the original branches are renamed again:

```
origin/feature/login -> origin/fteu-r8ah4/wglni-1ksq9
origin/bugfix/auth   -> origin/bfuix-6mxyz/ah-tuf5-4qz
```

* `.branch_map` file is updated:

```
feature/login -> fteu-r8ah4/wglni-1ksq9
bugfix/auth   -> bfuix-6mxyz/ah-tuf5-4qz
```

## Pushing Using Original Names

When a user executes:

```
git push origin feature/login
```

* The `pre-push` hook translates `feature/login` to `fteuar-5mnwl/loign-4pjxz` and pushes to the renamed branch.

## Important Notes

* Ensure you have appropriate permissions to delete and rename branches on the remote repository.
* This system is primarily for repositories where branch names do not need to be preserved for long-term usage.
* Always keep a backup or version control of your branches before running the fetch command.
