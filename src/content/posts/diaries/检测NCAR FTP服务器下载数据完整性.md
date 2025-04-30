---
# 必填
title: 检测NCAR FTP服务器下载数据完整性
published: 2024-02-03 11:47:48

# 可选
description: 每当运行CESM等气候模式，在配置好了`--compset`之后，往往需要花一段时间从NCAR的FTP数据服务器下载一些数据。假如下载时出现网络波动，会导致某些数据没有下载完全；或者，你不是在运行一个气候、气象模式，而只是需要拖下一大堆数据，那判断一大堆数据的完整性就显得十分重要了。
updated: 2024-02-03 11:47:48
tags:
  - CESM
  - Python

# 进阶，可选
draft: false
# pin: 0
toc: false
lang: zh
# abbrlink: theme-guide
---

每当运行CESM等气候模式，在配置好了`--compset`之后，往往需要花一段时间从NCAR的FTP数据服务器下载一些数据。假如下载时出现网络波动，会导致某些数据没有下载完全；或者，你不是在运行一个气候、气象模式，而只是需要拖下一大堆数据，那判断一大堆数据的完整性就显得十分重要了。一些博文（例如[HDF4(.nc)文件完整性检验](https://blog.csdn.net/qq_28491207/article/details/116177290)）通过与文件自描述数据中的文件大小比较来确定完整性。但尝试之后这个方法无法用于NetCDF。

所幸NCAR的FTP服务器提供了所有数据的MD5信息。利用这些信息可以写一个简单的脚本来检测数据的完整性。完整的脚本放在最后。脚本的最重要的函数包含两个部分。

## 遍历文件并计算MD5
首先遍历已经下载好的数据文件，对每个文件计算其MD5，之后将其存储为一个字典的`pickle`文件。

```Python
def init_from_inputdatalist():

    file_a = "Buildconf/cam.input_data_list"
    md5_values_a = {}
    with open(file_a, "r") as f:
        for line in f:
            parts = line.strip().split("=")
            if len(parts) == 2:
                path = parts[1].strip()
                name = path.split("/")[-1]
                if not name.endswith(".nc"):
                   continue
                try:                
                    with open(path, "rb") as file:
                        data = file.read()
                        md5 = hashlib.md5(data).hexdigest()
                        md5_values_a[name] = md5
               except:
                    print(f"Skip file {path}")
   with open("md5_values_a.pickle", "wb") as f:
        pickle.dump(md5_values_a, f)
```
## 将计算好的MD5与NCAR提供的MD5列表比较
之后，将计算好的每个文件的MD5值与NCAR提供的MD5列表进行比较：取每个文件的文件名获得正确的MD5，之后与计算的数据进行比较。如果数据MD5不一致则删除，并在最后输出一个需要重新下载的文件列表。

```python
def check_md5():
    with open("md5_values_a.pickle", "rb") as f:
        md5_values_a = pickle.load(f)

    print(md5_values_a)
    
    # 读取文件B，比较md5值和路径名
    file_b = "./inputdata_checksum.dat"
    
    with open(file_b, "r") as f:    
        for line in f:
            parts = line.strip().split()
            if len(parts) == 8:
                md5 = parts[0]
                path = parts[-1]

                found_match = False
                for name in md5_values_a:
                    if name == path.split("/")[-1] and md5_values_a.get(name) != md5:
                        found_match = True
                    
                    if found_match:
                        del md5_values_a[name]
                   
                    if not found_match:
                        print(f"Error in {path}")
                        
    with open("md5_values_a.pickle", "wb") as f:
        pickle.dump(md5_values_a, f)

```

## 完整程序
```Python
'''
Date: 2023-10-03 16:38:41
LastEditors: Liren
LastEditTime: 2023-12-16 15:48:06
'''
import hashlib
import pickle
import glob
import sys
import threading

init_md5 = True
refresh_md5 = False

def init_from_inputlist(file_list):
    md5_values_a = {}
    for file_path in file_list:
        with open(file_path, 'rb') as file:
            file_content = file.read()
            md5_values_a[file_path] = hashlib.md5(file_content).hexdigest()

    with open('md5_values_a.pickle', 'wb') as handle:
        pickle.dump(md5_values_a, handle, protocol=pickle.HIGHEST_PROTOCOL)


def compute_md5(file_path, md5_values):
    with open(file_path, 'rb') as file:
        file_content = file.read()
        md5_values[file_path] = hashlib.md5(file_content).hexdigest()

def init_from_inputlist_multithreaded(file_list):
    threads = []
    md5_values = {}

    for file_path in file_list:
        thread = threading.Thread(target=compute_md5, args=(file_path, md5_values))
        thread.start()
        threads.append(thread)

    for thread in threads:
        thread.join()

    with open('md5_values_a.pickle', 'wb') as handle:
        pickle.dump(md5_values, handle, protocol=pickle.HIGHEST_PROTOCOL)


def init_from_inputdatalist():
# 读取文件A，计算每个文件的md5值
    file_a = "Buildconf/cam.input_data_list"
    
    md5_values_a = {}

    with open(file_a, "r") as f:
        for line in f:
            parts = line.strip().split("=")
            if len(parts) == 2:
                path = parts[1].strip()
                name = path.split("/")[-1]
                if not name.endswith(".nc"):
                    continue
                try:                
                    with open(path, "rb") as file:
                        data = file.read()
                        md5 = hashlib.md5(data).hexdigest()
                        md5_values_a[name] = md5
                except:
                    print(f"Skip file {path}")

    with open("md5_values_a.pickle", "wb") as f:
        pickle.dump(md5_values_a, f)

def check_md5():
    with open("md5_values_a.pickle", "rb") as f:
        md5_values_a = pickle.load(f)

    print(md5_values_a)
    
    # 读取文件B，比较md5值和路径名
    file_b = "inputdata_checksum.dat"
    
    with open(file_b, "r") as f:    
        for line in f:
            parts = line.strip().split()
            if len(parts) == 8:
                md5 = parts[0]
                path = parts[-1]

                found_match = False
                for name in md5_values_a:
                    if name == path.split("/")[-1] and md5_values_a.get(name) != md5:
                        found_match = True
                    
                    if found_match:
                        del md5_values_a[name]
                   
                    if not found_match:
                        print(f"Error in {path}")
                        
    with open("md5_values_a.pickle", "wb") as f:
        pickle.dump(md5_values_a, f)

if init_md5:
    print(sys.argv[1])
    if len(sys.argv) > 1 and sys.argv[1]:
        input_file_list = sys.argv[1:3] 
        init_from_inputlist(input_file_list)
    else:
        init_from_inputdatalist()

if refresh_md5:
    pass

check_md5()
```
