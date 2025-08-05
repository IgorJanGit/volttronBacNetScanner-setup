VOLTTRON Setup Guide (WSL + Poetry + pyenv)
This guide walks you through setting up VOLTTRON development environment using WSL (Ubuntu), pyenv, poetry, and cloning the required repositories.

🪟 Step 1: On Windows PowerShell (with WSL)
Install Ubuntu WSL instance:

powershell
Copy
Edit
wsl --install Ubuntu --name volttron-core17
🐧 Step 2: Inside Ubuntu (WSL shell)
Update packages and install dependencies:

bash
Copy
Edit
sudo apt update
sudo apt install -y build-essential curl libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev libffi-dev liblzma-dev python3-openssl git
Install pyenv:

bash
Copy
Edit
curl https://pyenv.run | bash

echo -e '\n# pyenv setup' >> ~/.bashrc
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'eval "$(pyenv init --path)"\neval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc
Install Python and create virtual environment:

bash
Copy
Edit
pyenv install 3.10.8
pyenv global 3.10.8

python -m venv ~/volttron/venv
source ~/volttron/venv/bin/activate

pip install --upgrade pip
pip install toml
🔧 Step 3: pyenv shell integration (optional but recommended)
bash
Copy
Edit
echo -e '\n# >>> pyenv setup >>>' >> ~/.bashrc
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init --path)"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc
echo '# <<< pyenv setup <<<' >> ~/.bashrc
📝 Step 4: Create and run Python setup script
Create the setup script:

bash
Copy
Edit
touch setup_volttron.py
ls setup_volttron.py
chmod +x setup_volttron.py
Then either:

bash
Copy
Edit
python setup_volttron.py
# or
python3 setup_volttron.py
📦 Step 5: Install Poetry
Install poetry:

bash
Copy
Edit
curl -sSL https://install.python-poetry.org | python3 -

export PATH="$HOME/.local/bin:$PATH"
poetry --version
🐍 setup_volttron.py Script
Below is the setup_volttron.py script to automate the cloning and dependency setup:

<details> <summary>📜 Click to expand `setup_volttron.py`</summary>
python
Copy
Edit
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

# Git repositories to install
GIT_REPOSITORIES = [
    ["https://github.com/eclipse-volttron/volttron-core.git", "main", True, 1],
    ["https://github.com/eclipse-volttron/volttron-lib-zmq.git", "develop", True, 2],
    ["https://github.com/eclipse-volttron/volttron-lib-auth.git", "main", True, 3],
    ["https://github.com/eclipse-volttron/volttron-lib-base-driver.git", "develop", True, 4],
    ["https://github.com/eclipse-volttron/volttron-platform-driver.git", "develop", True, 5],
    ["https://github.com/eclipse-volttron/volttron-listener.git", "develop", True, 6],
    ["https://github.com/eclipse-volttron/volttron-lib-fake-driver.git", "develop", True, 7],
    ["https://github.com/eclipse-volttron/bacnet-scan-tool.git", "develop", True, 8]
]

# [functions omitted for brevity — include them in your repo]
# Make sure to include:
# - find_next_instance_number
# - create_instance_directory
# - clone_repositories
# - get_dependency_names
# - update_pyproject_toml
# - update_volttron_core_toml
# - show_next_steps

def main():
    instance_num = find_next_instance_number()
    instance_path = create_instance_directory(instance_num)
    repo_paths = clone_repositories(instance_path)
    cloned_dependencies = get_dependency_names()

    print("\nUpdating pyproject.toml files to comment out duplicate dependencies...")
    for repo_name, repo_path in repo_paths.items():
        if repo_name != "volttron-core":
            update_pyproject_toml(repo_path, repo_paths, cloned_dependencies)

    volttron_core_path = update_volttron_core_toml(repo_paths)

    print(f"\nVOLTTRON environment setup complete in {instance_path}")
    if volttron_core_path:
        show_next_steps(volttron_core_path)
    else:
        print("Failed to set up volttron-core correctly. Please check the logs for errors.")

if __name__ == "__main__":
    main()
</details>
⚙️ Step 6: Install Volttron Dependencies via Poetry
bash
Copy
Edit
python3 setup_volttron.py

cd ~/volttron/volttron_instance_1/volttron-core
poetry install
🚀 Step 7: Start VOLTTRON (if not already running)
bash
Copy
Edit
export VOLTTRON_HOME=$(pwd)/volttron_home/
volttron -vv -l volttron.log &>/dev/null &
🧪 Step 8: Verify It’s Running
bash
Copy
Edit
ps aux | grep volttron
tail -n 30 volttron.log
🔁 Optional: Monitor Fake Driver Logs
Replace the path as needed:

bash
Copy
Edit
tail -f /home/igor/volttron/volttron_instance_1/volttron-lib-fake-driver/volttron.log
