---
title: 凯撒密码的加密解密
tags: [密码学,Python]
categories:  密码学
date: 2018-09-07 09:25:00
top: true
cover: true
	 
---

# 前言

 >凯撒密码作为一种最为古老的对称加密体制，在古罗马的时候都已经很流行，他的基本思想是：通过把字母移动一定的位数来实现加密和解密。明文中的所有字母都在字母表上向后（或向前）按照一个固定数目进行偏移后被替换成密文。例如，当偏移量是3的时候，所有的字母A将被替换成D，B变成E，以此类推X将变成A，Y变成B，Z变成C。由此可见，位数就是凯撒密码加密和解密的密钥。

<!--more-->
#  凯撒密码加密脚

- 交单的26次加密脚本

```python

#coding:utf-8
upperDict=['A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R','S','T','U','V','W','X','Y','Z']
lowerDict=['a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z']
def cesarWithLetter(ciphertext,offset):
    '''
    凯撒密码 :
        只转换字母(包括大写小写)
    参数 : 
        ciphertext : 明文
        offset : 偏移量
    '''
    result = ""
    for ch in ciphertext:
        if ch.isupper():
            result=result+upperDict[((upperDict.index(ch)+offset)%26)]
        elif ch.islower():
            result=result+lowerDict[((lowerDict.index(ch)+offset)%26)]
        elif ch.isdigit():
            result=result+ch
        else:
            result=result+ch
    return result

def printAllResult(ciphertext):
    '''
    打印所有偏移结果
    '''
    for i in range(len(upperDict)):
        print cesarWithLetter(ciphertext,i)

ciphertext=raw_input("Please input the words :")
printAllResult(ciphertext)

```


# 自动控制偏移位自动解密

```python

#-*-coding:utf-8-*-
__author__ = 007
__date__ = 2016 / 02 / 04


#==================================================================#
#         凯撒密码(caesar)是最早的代换密码,对称密码的一种                #
#   算法：将每个字母用字母表中它之后的第k个字母（称作位移值）替代            #
#==================================================================#

def encryption():
    str_raw = raw_input("请输入明文：")
    k = input("请输入位移值：")
    str_change = str_raw.lower()
    str_list = list(str_change)
    str_list_encry = str_list
    i = 0
    while i < len(str_list):
        if ord(str_list[i]) < 123-k:
            str_list_encry[i] = chr(ord(str_list[i]) + k)
        else:
            str_list_encry[i] = chr(ord(str_list[i]) + k - 26)
        i = i+1
    print "加密结果为："+"".join(str_list_encry)

def decryption():
    str_raw = raw_input("请输入密文：")
    k = input("请输入位移值：")
    str_change = str_raw.lower()
    str_list = list(str_change)
    str_list_decry = str_list
    i = 0
    while i < len(str_list):
        if ord(str_list[i]) >= 97+k:
            str_list_decry[i] = chr(ord(str_list[i]) - k)
        else:
            str_list_decry[i] = chr(ord(str_list[i]) + 26 - k)
        i = i+1
    print "解密结果为："+"".join(str_list_decry)

while True:
    print u"1. 加密"
    print u"2. 解密"
    choice = raw_input("请选择：")
    if choice == "1":
        encryption()
    elif choice == "2":
        decryption()
    else:
        print u"您的输入有误！"

#if __name__ == "__main__":
#    main
```
