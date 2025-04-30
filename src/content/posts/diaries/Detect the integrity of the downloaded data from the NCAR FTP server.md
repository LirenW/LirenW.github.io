---
# 必填
title: Detect the integrity of the downloaded data from the NCAR FTP server
published: 2024-02-03 11:47:48

# 可选
description: Whenever running climate models such as CESM, after configuring `--compset`, it often takes some time to download some data from the NCAR FTP data server. If there are network fluctuations during the download, it will cause some when data was not be downloaded completely; or, if you are not running a climate or meteorological model, but just need to download a large amount of data, then judging the integrity of a large amount of data becomes very important.
updated: 2024-02-03 11:47:48
tags:
  - CESM
  - Python

# 进阶，可选
draft: false
# pin: 0
toc: false
lang: en
# abbrlink: theme-guide
---

Whenever running climate models such as CESM, after configuring `--compset`, it often takes some time to download some data from the NCAR FTP data server. If there are network fluctuations during the download, it will cause some when data was not be downloaded completely; or, if you are not running a climate or meteorological model, but just need to download a large amount of data, then judging the integrity of a large amount of data becomes very important. 

Fortunately, the NCAR FTP server provides the MD5 information of all data. Using this information, a simple script can be written to detect the integrity of the data. The complete script is placed at the end. The most important function of the script consists of two parts. 

## Traverse the files and calculate the MD5
First, traverse the downloaded data files, calculate the MD5 of each file, and then store it as a `pickle` file of a dictionary. 

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

## Compare the calculated MD5 with the MD5 list provided by NCAR
Afterwards, compare the calculated MD5 value of each file with the MD5 list provided by NCAR: Take the file name of each file to obtain the correct MD5, and then compare it with the calculated data. If the data MD5 is inconsistent, delete it, and output a list of files that need to be re-downloaded at the end. 

```python
def check_md5():
    with open("md5_values_a.pickle", "rb") as f:
        md5_values_a = pickle.load(f)

    print(md5_values_a)
    
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

## Complete program  

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
# Read file A and calculate the md5 value of each file  
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
    
    # Read file B and compare the md5 values and path names  
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
