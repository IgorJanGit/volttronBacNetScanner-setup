Step 1: On Windows PowerShell (with WSL)
```
wsl --install Ubuntu --name volttron-core17

```



Step 2: Inside Ubuntu (WSL shell)
```

sudo apt update
sudo apt install -y build-essential curl libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev libffi-dev liblzma-dev python3-openssl git

# Install pyenv
curl https://pyenv.run | bash

echo -e '\n# pyenv setup' >> ~/.bashrc
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'eval "$(pyenv init --path)"\neval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc

# Python & venv
pyenv install 3.10.8
pyenv global 3.10.8
python -m venv ~/volttron/venv
source ~/volttron/venv/bin/activate

# Pip packages
pip install --upgrade pip
pip install toml




```


step 3 pyenv shell integration
```
echo -e '\n# >>> pyenv setup >>>' >> ~/.bashrc
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init --path)"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc
echo '# <<< pyenv setup <<<' >> ~/.bashrc


```

step 4 set up python setup script 
```

touch setup_volttron.py

# make sure the script is here
ls setup_volttron.py

# make sure it's executable
chmod +x setup_volttron.py

# optional but useful
python setup_volttron.py

python3 setup_volttron.py


```

step 5 install Poertry 
```

curl -sSL https://install.python-poetry.org | python3 -

export PATH="$HOME/.local/bin:$PATH"

poetry --version



```

```
#!/usr/bin/env python3

import os

import sys

import subprocess

import shutil

import re

import toml

from pathlib import Path

  

# Configuration

BASE_INSTANCE_NAME = "volttron_instance"

  

# Git repositories to install - format: [repo_url, branch_name, editable, install_order]

GIT_REPOSITORIES = [

    ["https://github.com/eclipse-volttron/volttron-core.git", "main", True, 1],

    ["https://github.com/eclipse-volttron/volttron-lib-zmq.git", "develop", True, 2],

    ["https://github.com/eclipse-volttron/volttron-lib-auth.git", "main", True, 3],

    ["https://github.com/eclipse-volttron/volttron-lib-base-driver.git", "develop", True, 4],

    ["https://github.com/eclipse-volttron/volttron-platform-driver.git", "develop", True, 5],

    ["https://github.com/eclipse-volttron/volttron-listener.git", "develop", True, 6],

    ["https://github.com/eclipse-volttron/volttron-lib-fake-driver.git", "develop", True, 7]

	[https://github.com/eclipse-volttron/bacnet-scan-tool.git", "develop", True, 8]

]

  
  
  

def find_next_instance_number():

    """Find the next available instance number."""

    instance_num = 1

    while os.path.exists(f"{BASE_INSTANCE_NAME}_{instance_num}"):

        print(f"{BASE_INSTANCE_NAME}_{instance_num} already exists")

        instance_num += 1

    return instance_num

  
  

def create_instance_directory(instance_num):

    """Create the instance directory."""

    instance_path = f"{BASE_INSTANCE_NAME}_{instance_num}"

    os.makedirs(instance_path)

    print(f"Created directory: {instance_path}")

    return instance_path

  
  

def clone_repositories(instance_path):

    """Clone all repositories into the instance directory."""

    # Sort repositories by install_order

    sorted_repos = sorted(GIT_REPOSITORIES, key=lambda x: x[3])

  

    # Dictionary to store repo names and paths

    repo_paths = {}

  

    for repo_url, branch, editable, _ in sorted_repos:

        # Extract the repo name from the URL

        repo_name = repo_url.split("/")[-1].replace(".git", "")

  

        # Set the target directory for cloning

        repo_dir = os.path.join(instance_path, repo_name)

  

        print(f"Cloning {repo_name} from {repo_url} (branch: {branch})...")

  

        try:

            # Clone the repository with the specified branch

            subprocess.run(

                ["git", "clone", "-b", branch, repo_url, repo_dir],

                check=True

            )

            repo_paths[repo_name] = repo_dir

            print(f"Successfully cloned {repo_name}")

        except subprocess.CalledProcessError as e:

            print(f"Error cloning {repo_name}: {e}")

  

    return repo_paths

  
  

def get_dependency_names():

    """Extract the package names from the repository list."""

    dependency_names = set()

    for repo_url, _, _, _ in GIT_REPOSITORIES:

        repo_name = repo_url.split("/")[-1].replace(".git", "")

        dependency_names.add(repo_name)

    return dependency_names

  
  

def update_pyproject_toml(repo_path, repo_paths, cloned_dependencies):

    """Update a repository's pyproject.toml file by commenting out dependencies that we've cloned."""

    toml_path = os.path.join(repo_path, "pyproject.toml")

    if not os.path.exists(toml_path):

        print(f"No pyproject.toml found in {repo_path}")

        return

  

    # Create a backup

    backup_path = f"{toml_path}.bak"

    shutil.copy2(toml_path, backup_path)

    print(f"Created backup of {toml_path}")

  

    try:

        # Read the file line by line

        with open(toml_path, 'r') as f:

            lines = f.readlines()

  

        in_dependencies_section = False

        modified_lines = []

  

        for line in lines:

            # Check if we're entering the dependencies section

            if line.strip() == "[tool.poetry.dependencies]":

                in_dependencies_section = True

                modified_lines.append(line)

                continue

  

            # Check if we're leaving the dependencies section

            if in_dependencies_section and line.strip().startswith("["):

                in_dependencies_section = False

                modified_lines.append(line)

                continue

  

            # If in dependencies section, check if this is a dependency we've cloned

            if in_dependencies_section:

                # Look for lines like "volttron-lib-base-driver = ">=2.0.0rc0"" or similar

                for dep_name in cloned_dependencies:

                    # Skip if the repo's name itself is mentioned (we don't comment out self-references)

                    if os.path.basename(repo_path) == dep_name:

                        continue

  

                    # Only comment out if it's a dependency line (not a comment, etc.)

                    pattern = rf'^\s*{dep_name}\s*='

                    if re.match(pattern, line) and not line.strip().startswith('#'):

                        line = f"#{line}"

                        print(f"Commented out dependency {dep_name} in {toml_path}")

                        break

  

            modified_lines.append(line)

  

        # Write the updated file

        with open(toml_path, 'w') as f:

            f.writelines(modified_lines)

  

        print(f"Updated {toml_path} by commenting out cloned dependencies")

  

    except Exception as e:

        print(f"Error updating {toml_path}: {e}")

        # Restore the backup

        shutil.copy2(backup_path, toml_path)

        print(f"Restored backup due to error")

  
  

def update_volttron_core_toml(repo_paths):

    """Update the pyproject.toml file in volttron-core to include paths to other repositories."""

    volttron_core_path = repo_paths.get("volttron-core")

    if not volttron_core_path:

        print("Error: volttron-core repository not found.")

        return None

  

    pyproject_toml_path = os.path.join(volttron_core_path, "pyproject.toml")

    if not os.path.exists(pyproject_toml_path):

        print(f"Error: pyproject.toml file not found at {pyproject_toml_path}.")

        return None

  

    # Back up the original toml file

    backup_path = f"{pyproject_toml_path}.bak"

    shutil.copy2(pyproject_toml_path, backup_path)

    print(f"Created backup of pyproject.toml at {backup_path}")

  

    try:

        # Read the current content

        with open(pyproject_toml_path, 'r') as f:

            content = f.read()

  

        # Find the end of the [tool.poetry.dependencies] section

        dependencies_section = re.search(r'\[tool\.poetry\.dependencies\](.*?)(?=\[tool\.poetry|$)',

                                         content, re.DOTALL)

        if not dependencies_section:

            print("Could not locate dependencies section in pyproject.toml")

            return None

  

        # Extract the existing dependencies

        deps_content = dependencies_section.group(1)

  

        # Generate new dependency entries

        new_deps = []

        for repo_name, repo_path in repo_paths.items():

            # Skip volttron-core itself

            if repo_name == "volttron-core":

                continue

  

            # Create a relative path from volttron-core to the repo

            rel_path = os.path.relpath(repo_path, volttron_core_path)

            new_deps.append(f'{repo_name} = {{ path = "{rel_path}", develop = true }}')

  

        # Construct updated content

        before_deps = content[:dependencies_section.start() + len('[tool.poetry.dependencies]')]

        deps_with_new = deps_content.rstrip() + '\n' + '\n'.join(new_deps) + '\n'

        after_deps = content[dependencies_section.end():]

  

        updated_content = before_deps + deps_with_new + after_deps

  

        # Write the updated content

        with open(pyproject_toml_path, 'w') as f:

            f.write(updated_content)

  

        print(f"Updated {pyproject_toml_path} with local repository paths")

  

    except Exception as e:

        print(f"Error updating pyproject.toml: {e}")

        # Restore the backup on error

        shutil.copy2(backup_path, pyproject_toml_path)

        print(f"Restored backup due to error")

        return None

  

    return volttron_core_path

  
  

def show_next_steps(volttron_core_path):

    """Display the next steps to follow after setup."""

    print("\n" + "=" * 50)

    print("NEXT STEPS:")

    print("=" * 50)

    print(f"1. cd {volttron_core_path}")

    print("2. poetry install")

    print("3. export VOLTTRON_HOME=$(pwd)/volttron_home/")

    print("4. volttron -vv -l volttron.log &>/dev/null &")

    print("=" * 50)

  
  

def main():

    """Main function to set up the VOLTTRON environment."""

    # Find the next available instance number

    instance_num = find_next_instance_number()

  

    # Create the instance directory

    instance_path = create_instance_directory(instance_num)

  

    # Clone repositories and get their paths

    repo_paths = clone_repositories(instance_path)

  

    # Get the names of dependencies we've cloned

    cloned_dependencies = get_dependency_names()

  

    # Comment out matching dependencies in all pyproject.toml files

    print("\nUpdating pyproject.toml files to comment out duplicate dependencies...")

    for repo_name, repo_path in repo_paths.items():

        if repo_name != "volttron-core":  # We'll handle volttron-core separately

            update_pyproject_toml(repo_path, repo_paths, cloned_dependencies)

  

    # Update volttron-core's toml file and get its path

    volttron_core_path = update_volttron_core_toml(repo_paths)

  

    print(f"\nVOLTTRON environment setup complete in {instance_path}")

  

    # Show next steps

    if volttron_core_path:

        show_next_steps(volttron_core_path)

    else:

        print("Failed to set up volttron-core correctly. Please check the logs for errors.")

  
  

if __name__ == "__main__":

    main()


```



step 6 Install Volttron Dependencies via Poetry
```
python3 setup_volttron.py

cd ~/volttron-dev/volttron_instance_1/volttron-core
poetry install


```

step 7 
[if it  runing and shutting down do the export command ]
```
export VOLTTRON_HOME=$(pwd)/volttron_home/
volttron -vv -l volttron.log &>/dev/null &


```

step 8
```

ps aux | grep volttron
tail -n 30 volttron.log


```



for fake driver to watch 
```
tail -f /home/igor/volttron/volttron_instance_1/volttron-lib-fake-driver/volttron.log
```


```
```
