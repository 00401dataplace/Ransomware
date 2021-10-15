---
title : 02_Windows_目錄處理與檔案處理
---
# Windows_目錄處理與檔案處理
By Cheng-Yen, Tsai
Date : 2021-05-22

## 壹、前言
基本的目錄處理也就是對該目錄下的文件和子目錄進行操作，包括常用的複製、移動等等，針對這些行為，微軟也提供了蠻多函式可以使用。目錄處理是針對檔案的位置、檔案屬性、檔案的增刪等等進行操作，而檔案處理是針對檔案內部的內容作讀取、寫入等動作，這些對勒索病毒來說是很重要的操作。

---
## 貳、目錄處理的API
### 01. CopyFile()
* 會在同個檔案位置新增一份一模一樣的檔案，連最後修改時間等等都相同。
* 一共有三個參數。第一個是要被複製的檔案名稱，並且要給檔案位，若是不存在的檔案命稱，則產生錯誤**ERROR_FILE_NOT_FOUND**。
* 第二個是新檔案的名稱，同樣要給檔案位置。
* 最後是一個布林值，如果是False，且新檔名存在，則會直接覆蓋過去，但是如果存在的檔案屬性是「唯讀」或是「隱藏」，就會無法覆蓋，產生**ERROR_ACCESS_DENIED**錯誤。如果是True，則檔案不會改變，並產生**ERROR_FILE_EXISTS**錯誤。
* 下面是一個簡單複製檔案的例子: 
```clike=
#include <windows.h>
#include <stdio.h>
#include <tchar.h>
int main() {
	if (!CopyFile(_T("C:\\Users\\ASUS\\file.txt"),
		_T("C:\\Users\\ASUS\\file01.txt"), false)) {
		_tprintf(_T("CopyFile() error: %d\n"), GetLastError());
		return false;
	}
}
```
<div style="text-align: center">
<img src="https://i.imgur.com/KFIt3EE.png"/>
</div>

* 上表是在說明新舊檔案間的時間關係，在閱讀資料的同時，我們得知這是一個很關鍵的性質，因為許多軟體是用最後修改時間來判斷檔案是否被修改，所以在進行此項操作要留意這個性質。
* 另外與CopyFile()使用方法相近的有DeleteFile()、MoveFile()等等。
___
### 02.Delete()

* 刪除檔案，從目錄檔案的資訊從表中移除，檔案所佔的磁碟機空間，會被標上「未使用」的記號，而內容還是會留在磁碟機裡原本的位置。
```cpp=
if(!DeleteFile(_T("C:\\TEMP\\file2.txt")))
{
 _tprintf(_T("CopyFile() Error: %d\n"),
   GetLastError());
   return false;
}

```

---
### 03.MoveFile

* 移動檔案，是不會改變磁碟裡的空間，過程：將「檔案Ｇ」從「目錄Ａ」移至「目錄Ｂ」也就是將原檔案從目錄Ａ移除；但如果檔案Ａ與檔案Ｂ在不同的磁碟，就必須要有複製動作，也就是CopyFile()加上DeleteFile()。
```cpp=
if (!MoveFile(_T("C:\\TEMP\\OldName.txt"),
              _T("C:\\TEMP\\NewName.txt")
              ))
              {
              _tprintf(_T("CopyFile() Error: %d\n"),
               GetLastError());
               return false;
              }

```
___
### 04. GetFileAttributes()
* 在CopyFile()函式第三個三數中，已經有關於檔案屬性了，檔案屬性可能造成不同的結果甚至產生錯誤，所以在進行檔案的操作前，要先取得檔案的屬性。
* 以下是檔案屬性表 : 
<div style="text-align: center">

| 屬性                      | 實際數值     | 意義     | 
| ------------------------ | --------    | ------  |
| FILE_ATTRIBUTE_READONLY  | 1(0x1)      | 唯讀檔案 |
| FILE_ATTRIBUTE_HIDDEN    | 2(0x2)      | 隱藏檔案 |
| FILE_ATTRIBUTE_SYSTEM    | 4(0x4)      | 系統檔案 |
| FILE_ATTRIBUTE_ARCHIVE   | 32(0x20)    | 封存檔案 |
| FILE_ATTRIBUTE_NORMAL    | 128(0x80)   | 普通檔案 |
| FILE_ATTRIBUTE_TEMPORARY |256(0x100)   | 暫存檔案 |
| FILE_ATTRIBUTE_OFFLINE   |4096(0x1000) | 離線檔案 |
| FILE_ATTRIBUTE_ENCRYPTED |16384(0x4000)| 加密檔案 |

</div>


```clike=
#include <iostream>
#include <windows.h>
#include <string>
using namespace std;

int main() {
	DWORD T;
	LPCSTR T1 = "test01.txt";
	T = GetFileAttributesA(T1);
	cout << T;
}
```
___
### 05. SetFileAttributes()
```clike=
#include <iostream>
#include <windows.h>
#include <string>
using namespace std;

int main() {
	LPCSTR T = "test01.txt";
	SetFileAttributesA(T, 1);
}
```
## 參、檔案處理
### 01. CreateFile()
* 開啟檔案，系統會在記憶體裡存放結構，結構包含關於檔案的資料，並且回傳一個HANDLE，之後對於這個檔案的操作就透過這個HANDLE進行。
* 在官網上看到CreateFile()的宣告使用七個參數:
  * **LPCTSTP lpFileName** => 指定檔案名稱，但是不能是目錄。
  * **DWORD   dwDesiredAccess** => 指定開啟模式，可以是:
                            GENERIC_READ : 可以讀取，不能寫入
                            GENERIC_WRITE : 可以寫入，不能讀取
                            GENERIC_READ | GENERIC_WRITE : 寫入並且讀取
                            0 : 不讀取也不寫入
  * **DWORD dwShareMode** => 檔案開啟後，是否允許別的程式對此檔案的動作，可以是:
                            0 : 完全不允許其他程式開啟此檔案
                            FILE_SHARE_DELETE : 允許檔案被刪除或更改檔名。
                            FILE_SHARE_READ : 允許別的程式以GENERIC_READ模式開啟
                            FILE_SHARE_WRITE : 允許別的程式以GENERIC_WRITE模式開啟
  * **LPSECURITY_ATTRIBUTE** => 此參數是關於資訊安全的，可以設為NULL，使用預設值。
  * **DWORD CreationDisposition** => 關於檔案存在與不存在的動作，可以是:
    1. CREATE_NEW : 若不存在，則建立新檔案，存在則回傳  ERROR_FILE_EXISTS錯誤
    2. CREAT_ALWAYS : 若不存在，則建立新檔案，存在且有寫入權限，則清除所有內容，逤存在且沒有寫入權限，則回傳ERROR_ALREADY_EXIST錯誤
    3. OPEN_EXISTING : 開啟存在的檔案，若不存在，則回傳ERROR_FILE_NOT_FOUND錯誤
    4. OPEN_ALWAYS : 若檔案不存在，則建立檔案，檔案存在且有寫入權限，則開啟檔案且不更動內容，若無權限，則回傳ERROR_ALREADYEXISTS錯誤
    5. TRUNCATE_EXISTING : 開啟存在的檔案，且清空內容，若檔案不存在，則回傳ERROR_FILE_NOT_FOUND錯誤
  * **DWORD dwFlagsAndAttributes** => 設定檔案屬性，只對產生新檔案有效，至於已存在的檔案，會忽落這個參數，其中參數值就是上方的檔案屬性表
  * **HANDLE hTemplateFile** : 此參數目前還沒觀察到特定的效果

* 另外針對回傳值也要檢查，回傳值是一個HANDLE，檔案不一定每次都開啟成功，上述列出許多可能發生的錯誤，但是CreateFile()回傳的是INVALID_HANDLE_VALUE，透過此錯誤訊息，並沒辦法確認實際的出錯點在哪，所以要透過GetLastError()來取得錯誤碼，根據錯誤碼找出真正的出錯原因。
* 

