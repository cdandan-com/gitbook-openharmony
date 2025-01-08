# 12-图形子系统-GPU适配

## 架构

<figure><img src=".gitbook/assets/渲染GPU架构图 (1).png" alt=""><figcaption></figcaption></figure>

其中native window转换并非必须，依照其他操作系统接口转换，则需要native window转换，将EGL接口，转换到OpenHarmony系统上使用。

