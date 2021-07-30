# dlib bug

更新 win11 及 cuda 后在 python 中`import dlib`报错
```bash
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "C:\Users\Lc\AppData\Local\Programs\Python\Python37\lib\site-packages\dlib\__init__.py", line 19, in <module>
    from _dlib_pybind11 import *
ImportError: DLL load failed: 找不到指定的模块。
```
用 pip 切换 dlib 版本时 build 报错
```bash
AttributeError: type object 'Callable' has no attribute '_abc_registry'
```
首先 `pip uninstall typing` 解决上述报错，变为下面这个报错
```bash
    -- Could NOT find Boost
    -- Found PythonLibs: C:/Users/Lc/AppData/Local/Programs/Python/Python37/libs/python37.lib (found suitable version "3.7.0", minimum required is "3.4")
    --  *****************************************************************************************************
    -- We couldn't find the right version of boost python. 
```
通过 `pip install -U cmake` 解决上述报错。
最后 `pip install --force-reinstall dlib` 成功。