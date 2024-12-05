1. pip安装selenium

2. 安装google-chrome

    ```bash
    wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb -o chrome.deb
    apt install ./chrome.deb
    ```

3. 运行`google-chrome --version`，查看当前chrome版本
4. 从`https://googlechromelabs.github.io/chrome-for-testing/`下载版本相近的driver
5. 解压后放置到特定目录
6. selenium中通过以下语句指定
7. 
    ```python
    webdriver.Chrome(options=chrome_options,executable_path='/path/to/chromedriver')
    ```
