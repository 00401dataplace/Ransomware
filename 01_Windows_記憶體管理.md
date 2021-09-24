---
title : 01_Windwos_記憶體管理
---

# Windows_記憶體管理
By Cheng-Yen, Tsai
Date : 2021-05-21

---
## 壹、前言
在初步了解記憶體的運作原理後，我開始著手進行記憶體管理，開始操作簡單的記憶體配置、記憶體內容複製等等，目標是達到在windows靈活的操控記憶體。

---
## 貳、記憶體配置
### 01. 所需知識
* 堆積(Heap) : 因為每筆資料所需的空間都不一定相同，所以不可以對所有資料都配置相同大小的記憶體，所以便產生了heap這個記憶體空間。**此空間在得到配置請求時，會回傳一個指標。**
* 在windows中要進行動態記憶體配置，可以使用HeapAlloc()。如同C語言的malloc()，只是參數上的差別，在windows上，也可以使用malloc()加上free()來達到相同的效果，但是使用windows本身的API能夠做到更細部的控制。
* 使用HeapAlloc()有幾個條件，首先要先取的對heap的handle，簡單的做法是使用GetProcessHeap()，若成功，則回傳這個heap，失敗則回傳NULL。
* 然而，HeapAlloc()含有另外兩個參數，一個是旗標，在這裡我都設定成0(一般都是這樣)，雖然還有兩三種選擇，但屬於更深入的部分了。另一個參數就是要配置的記憶體數量。
* 同樣的，若成功配置記憶體，則回傳配置好的記憶體位址，否則回傳NULL。
* 配置完記憶體後，接著要釋放記憶體，使用HeapFree()來操作，同樣的先取得heap的hendle，使用GetProcessHeap()，參數也有旗標(設定成0)，最後是要釋放的記憶體。
* 下面是一個複製字串的程式:
```clike=
#include <windows.h>
#include <stdio.h>

int main() {
	char aString[] = "Hello, world!";
	int slen = strlen(aString);
	char* pString = NULL;
	printf("aString: [%s]\n", aString);
	pString = (char *)HeapAlloc(GetProcessHeap(), 0, slen + 1);
	if (NULL == pString){
		printf("HeapAlloc() error\n");
		return 1;
	}
	strcpy_s(pString, slen + 1, aString);
	printf("pString: [%s]\n", pString);
	HeapFree(GetProcessHeap(), 0, pString);
}
```
### 02. 困難點
* 在剛開始學習的過程，我無法理解為何需要宣告一個pString指標，直到我回頭查看資料才發現，在heap收到配置請求時，所回傳的是一個指標，所以當然需要宣告這個指標來操作。
* 在第14行的strcpy_s()，我也摸索了很久，之後在mincrosoft的文件中看到，是為了確保程式的安全性，後面接的s是safe的意思，在許多記憶體相關的函式都有此設定，在第一哥參數後面加上了長度的參數，代表目標記憶體大小，這樣可以避免記憶體過度的使用造成程式的錯誤和漏洞。
* HeapFree()的最後一個參數是指標，其原因也是一樣，對記憶體的操作要透過回傳的指標。
---

## 參、常用的記憶體函式
### 01. CopyMemory()
* Windows內使用CopyMemory()來搬動記憶體內容，相當於C語言裡的memcpy()。
* 其中有三個參數，第一個是目標位址，第二個是來源位址，最後是複製的記憶體數量，單位是byte。
* 下面是複製記憶體內容(字串)的程式:
```clike=
#include <windows.h>
#include <stdio.h>

int main() {
	char aString1[] = "Hello, world!";
	char aString2[100];
	printf("aString1: [%s]\n", aString1);
	CopyMemory(aString2, aString1, strlen(aString1) + 1);
	printf("aString2: [%s]\n", aString2);
	return 0;
}
```
![](https://i.imgur.com/wjBeL20.png)

* 剛開始我漏掉了最後的記憶體數量要加1的部分，原因是字串最後會有一個空字元。
* 另外來源跟目標是可以重疊的:
```clike=
#include <windows.h>
#include <stdio.h>

int main() {
	char aString1[1024] = "Hello, world!";
	printf("%s\n", aString1);
	CopyMemory(aString1+7, aString1, strlen(aString1));
	printf("%s\n", aString1);
	return 0;
}
```
![](https://i.imgur.com/sDzeapx.png)


### 02. FillMemory() 與 ZeroMemory()
* 這兩個函式式用來設定記憶體內容的，前者是用來將一快記憶體內填滿指定的字元，等於C語言中的memset()。後者是將記憶體內填滿0，但其實我認為有點多此一舉，同樣的FillMemory()也可以做到。
* FillMemory()有三個參數，第一個是目標為址，第二個是填充的數量，最後是要用哪個字元填充。
* ZeroMemory()就只有前兩個參數設定。
* 下面是兩個函式的使用:
```clike=
#include <windows.h>
#include <stdio.h>

int main() {
	char aString[] = "Hello, world!";
	printf("aString: [%s]\n", aString);
	FillMemory(aString, strlen(aString), '@');
	printf("aString: [%s]\n", aString);
}
```
![](https://i.imgur.com/wYgRuXP.png)
* 使用Fillmemory()替代Zeromemory()
```clike=
#include <windows.h>
#include <stdio.h>

int main() {
	char aString[] = "Hello, world!";
	printf("aString: [%s]\n", aString);
	FillMemory(aString, strlen(aString), '\0');
	printf("aString: [%s]\n", aString);
}
```
![](https://i.imgur.com/3fd8jqu.png)
