------------------------------------------------------------------------
    Setting Up Spark Standalone on Docker (Windows) 
------------------------------------------------------------------------

Repo的使用教程

前提：有什么看不懂的，或者内容不够完整，可以直接问AI。

确保windows：
    1. 下载 docker desktop
    2. 下载 wsl2
        检查: wsl --version
    3. 下载 ubuntu发行版 （我使用 Ubuntu-24.04版本）
        1. 点击vscode右下角按钮
        2. Connect to WSL using Distro...
        3. 下载 Ubuntu-24.04
        4. 会跳出 Ubuntu terminal窗口
        5. 要求输入username和password
            username = 会显示在每一行terminal cmd前面，以及/home/username
            password = 是以后使用sudo cmd要用到的
        6. 完成后 exit窗口
    4. 到 docker desktop 打开 ubuntu的integration （让 ubuntu能够使用docker）
    5. 设置 ubuntu 为 default wsl （默认是docker-desktop）
        查看 default wsl，输入:
            wsl --status
        或者
            wsl -l -v
        更改默认:
            wsl --set-default <Distro-Name>

    补充
        如果已经下载 docker desktop，
        会发现 wsl -l -v 已经有一个docker-desktop，这是docker的linux vm
        docker-desktop不是我们直接进入和使用的，而是我们通过在terminal用docker cmd来交互的


Enter Ubuntu and Create Project
    在vscode，进入wsl2 ubuntu模式
    ls 查看当前位置（/home/username）

    这个ubuntu将会是我们以后用来做所有需要linux环境的project，就如spark project的地方。
    不需要下载多个wsl2 linux os，只需要一个Ubuntu就能管理所有project。可以通过以下方式达成。

    我们会在/home/username开一个repo/（这是repositories）
    所有的projects就会放在这个/home/username/repo/

    这里有两种方式create project
    方式1 = git clone（这个方式不需要在repo开project folder）
    步骤
        cd /home/username/repo
        git clone https://github.com/junmob/spark-standalone.git

    方式2 = 想要自己build from scratch
    步骤
        cd /home/username/repo
        mkdir spark-standalone
        # 为每个file写入代码...
    如果大家想学习或者不了解代码运作原理，可以到docs/里面查看我写的markdown，附上英文和中文版本

    方式1 = 适合快速启动，跳过学习docker（DevOps），直接学习spark代码
    方式2 = 适合先学习docker，后学习spark代码

    补充
        当我们有其他projects时，就在repo下创建其他project folder
        例如 下一次要做setting up airflow on docker的project，
        我们就在/home/username/repo 创建 airflow-docker，这样所有projects都会在这里


Install python3.11 to Ubuntu
    接着，假设folder里的files齐全，
    我们要下载 python 3.11
    要求3.11，因为spark on docker也用3.11，相同版本或许可以减少bug出现的概率
    步骤
        sudo apt update
        sudo apt install python3.11 python3.11-venv

    install python3.11可能出现的bug：
        E: Unable to locate package python3.11
        E: Couldn't find any package by glob 'python3.11'
    这里需要额外的installation
    步骤
        sudo apt update
        sudo apt install software-properties-common -y
        sudo add-apt-repository ppa:deadsnakes/ppa -y
        sudo apt update
    回到之前的步骤
        install python3.11 python3.11-venv


Create and Activate Venv
    接着，我们要创建venv
    建议在每个project创建各自的venv，用来保存只属于这个project需要的dev tools，避免ubuntu杂乱
    步骤
        python3.11 -m venv .venv
        source .venv/bin/activate
    这时候terminal可以看到:
    (venv) username@hostname: ~/repo/spark-standalone$


Install Dev Tools
    venv创建好后，可以开始下载dev tools
    步骤
        pip install pip-tools
    这个pip-tools是给接下来compile requirements.in用的
    如果不知道什么是requirements.in可以到docs里面查看，我把它写在docs/01_Learn_Spark_Dockerfile.md


Compile requirements
    有了pip-tools，我们现在要compile requirements.in
    步骤
        pile requirements/requirements.in
    完成后，requirements/ folder会出现 requirements.txt


下载requirements.txt 到这个venv
    在spark_apps/ folder写 .py 的时候会用到，例如 语法检查，自动补全，字体高亮
    步骤
        pip install -r requirements/requirements.txt
    尽管代码是在container运行的，但是写ETL代码是在vscode进行的
    没下载的话vscode会看不懂代码，频繁出现红色波浪线
    除非只用jupyterlab来写代码（后面会提到怎么用jupyterlab）
    

下载 make到这个venv
    make是用来简化docker cmd的工具
    可以查看Makefile，里面都是能用来与docker交互的make cmd
    打开Makefile可以发现docker cmd很长，我们可以透过make简短cmd
    步骤
        sudo apt update
        sudo apt install build-essential -y
    补充
        build-essential 是一个“开发者全家桶”，包含了 make、gcc、g++ 等一系列编译工具。


下一步
    在build docker image之前，我们需要处理linux的用户权限问题
    步骤
        sudo groupadd docker
        sudo usermod -aG docker $USER
        newgrp docker
        source .venv/bin/activate
        docker ps
    说明
        假设docker ps没有报错，就说明我们的linux用户可以与docker交互了

Build Docker Image
    开始build docker image
    如果大家不知道每个make cmd怎么用，建议先了解再继续以下步骤。
    一般第一次build image会使用build-progress。
    步骤
        make build-progress

    build好后，可以写以下命令查看images list
        docker image ls

    确认是否有两个images出现：
        1. spark-jupyter (5.76GB)
        2. spark-image (5.48GB)

补充
    如果发现project folder里的某个代码有bug并且修改后，在启动docker container之前，需要再次make build
    没有build的话，container是根据之前build的image启动的，修改后的setting将不会被更新


Run a Docker Container
    所有工作都做好了，现在可以启动docker container。
    再次提醒，如果大家不知道每个make cmd怎么用，建议先了解再继续以下步骤。
    简单说明:
        1. 想普通使用，就用  make run -d
        2. 想体验有3个workers的spark，就用 make run-scaled 或者 make run-generated (better)
        3. 如果想运行前台模式和查看logs，就用 make run（不常用，前台模式很碍事）
    成功run container后，有几个services可以用:
        1. Master UI （localhost:9090）
        2. jupyterlab (localhost:8888)
        3. history server (localhost:18080)
        4. driver UI  (localhost:4040 for jupyter and localhost:4041 for spark_apps)


Important file and folder
    1. data/
        data是用来放raw data的，例如csv，parquet，excel
        它是volume mount的（同步local和container）
    2. notebooks/
        notebooks是用来放.ipynb的
        jupyterlab的notebooks统统save在这里
        jupyter提供探索data和试错的环境
        它是volume mount的（同步local和container）
    3. spark_apps/
        spark_apps是用来正式写 etl.py 的
        需要通过makefile中的 make submit app=xxx.py 来提交 .py 代码给 master
        它是volume mount的（同步local和container）
    4. Makefile
        查看有哪些short form docker cmd可以用
        或者可以根据自己常用的docker cmd来修改
    5. docs
        我写了很多我学习docker的过程，欢迎阅读。
        里面有Dockerfile，docker-compose,以及 Makefile的说明。


Reference
1. Setting up a Spark standalone cluster on Docker in layman terms (Marin Aglić, Jan 1, 2023)
    https://medium.com/@MarinAgli1/setting-up-a-spark-standalone-cluster-on-docker-in-layman-terms-8cbdc9fdd14b
    https://github.com/mrn-aglic/spark-standalone-cluster
2. Setting up Spark using Docker (Dimitris Kalouris, Fed 3, 2025)
    https://medium.com/@dkalouris/setting-up-spark-using-docker-59db2d073487
    https://github.com/dkalouris/sparkpg/tree/medium_tutorial
3. How to build an automated data pipeline using Airflow, dbt, Postgres, and Superset (Windows 11 WSL)
    https://www.youtube.com/watch?v=vMgFadPxOLk&list=PLK4m6f5yKgyzgRkF2uupDjRMzDU3_-A1H&index=15&t=483s


免责声明
    本人纯属分享学到的知识，并非专业
    代码和知识来自references以及 AI(ChatGPT, Deepseek, Gemini Pro) 给的知识