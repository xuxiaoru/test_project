在使用 GitHub LFS（Large File Storage）上传文件后，可以通过以下步骤下载这些文件：

1. **安装 Git LFS：**
   确保你本地已经安装了 Git LFS。如果还没有安装，可以通过以下命令安装：
   
   ```bash
   git lfs install
   ```

2. **克隆包含 LFS 文件的仓库：**
   使用 `git clone` 命令克隆仓库，Git LFS 会自动下载 LFS 文件。

   ```bash
   git clone https://github.com/username/repository.git
   ```

3. **下载 LFS 文件：**
   当你克隆了仓库之后，Git 只会下载 LFS 文件的占位符（指针文件）。要下载实际的文件，可以运行以下命令：

   ```bash
   git lfs pull
   ```

4. **手动获取单个文件：**
   如果你只需要下载某些特定的 LFS 文件，可以使用如下命令获取：
   
   ```bash
   git lfs fetch
   git lfs checkout <file_path>
   ```

这几步就可以让你从 GitHub LFS 下载你需要的文件。如果你想要确保文件在克隆时自动下载，可以在克隆仓库之后立即运行 `git lfs pull` 来获取所有 LFS 文件。