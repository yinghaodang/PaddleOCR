# 魔改的PaddleOCR

对应官方代码仓库 On branch release/2.6rc

起因是为了在 python3.11.4 上跑 PaddleOCR。但是由于python版本太新，导致使用pip安装的时候经常遇到 PyMuPDF 模块报错。

PyMuPDF 支持 python311 的版本 被2.7版本的 PaddleOCR 毙掉了。PaddleOCR 推荐的 PyMuPDF 版本不支持 python311.

精挑细选安装了 `pip install paddleocr==2.6.0.1 --upgrade PyMuPDF==1.21.1`

使用过程中有三点不舒适

1. PaddleOCR 的日志会混入 StarRailAutoProxy 的日志，于是我删除了源码中的部分日志

2. PaddleOCR 的后处理有bug, ppocr\postprocess\db_postprocess.py 的 188-191 行的写法，在低版本numpy上是 warning，目前已经直接 error. 修改方法： `np.int -> np.int32`

3. PaddleOCR 的后处理有bug, ppocr\postprocess\rec_postprocess.py 的 85 行左右，存在推理 index 超出 字符集 的 bug.
    ```
    # origin
    char_list = [
        self.character[text_id]
        for text_id in text_index[batch_idx][selection]
    ]
    -------------
    # changed
    char_list = [
        self.character[text_id]
        for text_id in text_index[batch_idx][selection] if text_id < len(self.character)
    ]
    ```
4. 上面三点是paddleocr 2.6.0.1 上的bug, 但是 gitub 上不知道哪个分支才是 2.6.0.1 ，下载了 release/2.6rc 这个分支。

   事实上，这个分支对应的是 2.6.0.3 版本的 paddleocr，我在测试过程中还修改了ppocr\postprocess\rec_postprocess.py 的 119-122 行：
   ```
    # origin
    if abs(num_boxes - 2) < 1e-4:
        sorted_boxes = sorted(dt_boxes, key=lambda x: (x[1], x[0]))
    else:
        sorted_boxes = sorted(dt_boxes, key=lambda x: (x[0][1], x[0][0]))
    ------------------
    # changed
    sorted_boxes = sorted(dt_boxes, key=lambda x: (x[0][1], x[0][0]))
   ```
## 编译方法

```
python setup.py bdist_wheel
pip3 install dist/paddleocr-x.x.x-py3-none-any.whl # x.x.x是paddleocr的版本号
```
也可以使用我编译好的whl文件，在dist文件夹下。

## 其他

1. 本人测试环境是 Win11 ，conda虚拟环境中编译，速度很快，如果卡死可能是网络问题。

2. 在编译的过程中没有遇到PyMuPDF这个依赖，也就是说没有这个模块也是可以运行的（只要不去推理PDF），那么为什么安装的时候一定要我装这个呢？还是说pip install 有什么参数可以避开某个依赖？ 是--no-deps吗？有没有更近一步，只屏蔽部分依赖的方法？

3. 安装过程，在虚拟环境中一切正常（miniconda产生），但是我还有另一个环境，是微软商店下载的Python。~~当时图个省力。后来还是没用习惯。~~ 这个环境遇到了 `[WinError 5] 拒绝访问` 这个错误，我确信我是以管理员权限运行的，也试过网上的解决方案，加上`--user`，不起作用。然后我猜到这是个 Windows 限定的bug，重启可能解决问题。于是，问题解决了。
