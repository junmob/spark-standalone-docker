# Setting Up Spark Standalone on Docker (Windows)

## Tutorial for Using this Repository

**Prerequisite:** If anything is unclear or seems incomplete, feel free to ask an AI assistant directly.

### Ensure your Windows system has:
1.  Docker Desktop installed.
2.  WSL2 installed.
    *   Check: `wsl --version`
3.  A Ubuntu distribution installed (I used Ubuntu 24.04).
    1.  Click the button in the bottom-right corner of VSCode.
    2.  Select "Connect to WSL using Distro...".
    3.  Download Ubuntu-24.04.
    4.  A Ubuntu terminal window will pop up.
    5.  Set up a `username` and `password`.
        *   `username` = Will be displayed at the start of each terminal command line and in `/home/username`.
        *   `password` = Required later for using `sudo` commands.
    6.  Close the window with `exit` once done.
4.  Enable Ubuntu integration in Docker Desktop (to allow Ubuntu to use Docker).
5.  Set Ubuntu as the default WSL distribution (default is `docker-desktop`).
    *   Check the default WSL distro:
        ```bash
        wsl --status
        # or
        wsl -l -v
        ```
    *   Change the default:
        ```bash
        wsl --set-default <Distro-Name>
        ```

**Additional Note:**
If Docker Desktop is already installed, running `wsl -l -v` will show a `docker-desktop` distribution. This is Docker's Linux VM. We don't enter or use `docker-desktop` directly; we interact with it through `docker` commands in the terminal.

---

### Enter Ubuntu and Create the Project
In VSCode, switch to WSL2 Ubuntu mode.
Run `ls` to see your current location (`/home/username`).

This Ubuntu instance will be our workspace for all projects requiring a Linux environment, like this Spark project. You don't need multiple WSL2 Linux OS installations; one Ubuntu can manage all projects by organizing them as follows.

We will create a `repo/` directory in `/home/username/` (for "repositories").
All projects will reside inside `/home/username/repo/`.

There are two ways to create the project:

**Method 1: `git clone`** (This method doesn't require manually creating a project folder inside `repo/`)
```bash
cd /home/username/repo
git clone https://github.com/junmob/spark-standalone.git
```

**Method 2: Build from scratch** (If you want to learn)
```bash
cd /home/username/repo
mkdir spark-standalone
# Write code for each file...
```
If you want to learn or don't understand how the code works, check the markdown files in `docs/`, which include both English and Chinese versions.

*   **Method 1** is suitable for quickly getting started, skipping the setup learning curve and jumping straight into Spark development.
*   **Method 2** is suitable for learning Docker first, then Spark code.

**Additional Note:**
When we have other projects, we simply create another project folder under `repo/`.
For example, for a future "setting up Airflow on Docker" project, we would create `airflow-docker` in `/home/username/repo`. This keeps all projects organized.

---

### Install Python 3.11 in Ubuntu
Assuming the project folder and files are ready, we need to install Python 3.11.
Version 3.11 is required because Spark on Docker in my project also uses 3.11. Matching versions may reduce the chance of bugs.
```bash
sudo apt update
sudo apt install python3.11 python3.11-venv
```

**Troubleshoorting: if `install python3.11` fails:**
```
E: Unable to locate package python3.11
E: Couldn't find any package by glob 'python3.11'
```
This requires an additional installation step:
```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
```
Then return to the previous step:
```bash
sudo apt install python3.11 python3.11-venv
```

---

### Create and Activate a Virtual Environment
Next, create a virtual environment (`venv`).
It's recommended to create a separate `venv` for each project to isolate the required development tools and avoid cluttering the main Ubuntu system.
```bash
python3.11 -m venv .venv
source .venv/bin/activate
```
Your terminal prompt should now look like:
`(venv) username@hostname: ~/repo/spark-standalone$`

---

### Install Development Tools
With the `venv` activated, install development tools.
```bash
pip install pip-tools
```
`pip-tools` is used to compile `requirements.in`.
If you are unfamiliar with `requirements.in`, please refer to `docs/01_Learn_Spark_Dockerfile.md` where I explain it.

---

### Compile Requirements
With `pip-tools` installed, compile `requirements.in`.
```bash
pip-compile requirements/requirements.in
```
After completion, a `requirements.txt` file will appear in the `requirements/` folder.

---

### Install Packages from `requirements.txt` into the Venv
This is needed when writing `.py` files in the `spark_apps/` folder for features like syntax checking, auto-completion, and syntax highlighting.
```bash
pip install -r requirements/requirements.txt
```
Even though the code runs inside a container, we write ETL code in VSCode.
Without these packages installed locally, VSCode won't recognize the libraries and will show "ImportError" or red underlines.  
(Unless you write code exclusively in JupyterLab, which will be mentioned later.)

---

### Install `make` (via build-essential) in the Venv
`make` is a tool used to simplify Docker commands.
Check the `Makefile` to see the available shorthand `make` commands for interacting with Docker. The Docker commands can be lengthy, and `make` helps shorten them.
```bash
sudo apt update
sudo apt install build-essential -y
```
**Additional Note:**
`build-essential` is a "developer toolkit" that includes `make`, `gcc`, `g++`, and other compilation tools.

---

### Next Step
Before building the Docker image, we need to handle Linux user permissions for Docker.
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
source .venv/bin/activate # Reactivate venv if needed
docker ps
```
**Explanation:**
If `docker ps` runs without errors, it means our Linux user can now interact with Docker.

---

### Build the Docker Image
Now, build the Docker image.
If you are unsure about the usage of each `make` command, it's recommended to understand them first before proceeding.
Typically, `build-progress` is used for the first build.
```bash
make build-progress
```
After building, list the Docker images to confirm:
```bash
docker image ls
```
You should see two images:
1.  `spark-jupyter` (~5.76GB)
2.  `spark-image` (~5.48GB)

**Additional Note:**
If you find a bug in the project folder's code and fix it, you need to run `make build` again *before* starting the Docker container.
Without rebuilding, the container will start from the previously built image, and your changes won't be reflected.

---

### Run a Docker Container
All preparations are complete. Now you can start the Docker container.
Again, ensure you understand the `make` commands.
A simple guide:
1.  For normal use with a single worker: `make run -d`
2.  To experience a Spark cluster with 3 workers: `make run-scaled` or `make run-generated` (better)
3.  To run in foreground mode and view logs: `make run` (Less common, as foreground mode can be intrusive)

After successfully running the container, several services become available:
1.  **Master UI:** `http://localhost:9090`
2.  **JupyterLab:** `http://localhost:8888`
3.  **History Server:** `http://localhost:18080`
4.  **Driver UI:** `http://localhost:4040` (for Jupyter) and `http://localhost:4041` (for `spark_apps`)

---

### Important Files and Folders
1.  **`data/`**
    *   Stores raw data like CSV, Parquet, Excel files.
    *   It is volume-mounted (synchronized between local host and container).
2.  **`notebooks/`**
    *   Stores `.ipynb` (Jupyter Notebook) files.
    *   All notebooks created in JupyterLab should be saved here.
    *   JupyterLab provides an environment for data exploration and experimentation.
    *   It is volume-mounted.
3.  **`spark_apps/`**
    *   For writing formal ETL `.py` scripts.
    *   Use the `make submit app=xxx.py` command from the `Makefile` to submit `.py` code to the Spark master.
    *   It is volume-mounted.
4.  **`Makefile`**
    *   Check for available short-form Docker commands.
    *   You can also modify it to add your own commonly used Docker commands.
5.  **`docs/`**
    *   Contains detailed notes on my learning process for Docker, Dockerfiles, `docker-compose`, and the `Makefile`. Welcome to read them.

---

### References
1.  **Setting up a Spark standalone cluster on Docker in layman terms** (Marin Aglić, Jan 1, 2023)
    *   Medium Article: https://medium.com/@MarinAgli1/setting-up-a-spark-standalone-cluster-on-docker-in-layman-terms-8cbdc9fdd14b
    *   GitHub: https://github.com/mrn-aglic/spark-standalone-cluster
2.  **Setting up Spark using Docker** (Dimitris Kalouris, Feb 3, 2025)
    *   Medium Article: https://medium.com/@dkalouris/setting-up-spark-using-docker-59db2d073487
    *   GitHub: https://github.com/dkalouris/sparkpg/tree/medium_tutorial
3.  **How to build an automated data pipeline using Airflow, dbt, Postgres, and Superset (Windows 11 WSL)** (YouTube Playlist)
    *   https://www.youtube.com/watch?v=vMgFadPxOLk&list=PLK4m6f5yKgyzgRkF2uupDjRMzDU3_-A1H&index=15&t=483s

---

**Disclaimer:**
This repository is a personal sharing of knowledge and is not professionally authored. The code and knowledge are derived from the references listed above and insights provided by AI assistants (ChatGPT, DeepSeek, Gemini Pro).